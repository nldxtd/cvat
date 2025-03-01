# CVAT Walk Through and Audio Implementation Approch

## Backend

### Backend walk through

The main code of backend lies in ./cvat/apps/engine/. The main models we will be using is these 4 models: `Task` `Segment` `Job` `Data`. The two main part I seeked is task creation and job data fetching.

#### Create of a task

The full task creation along with the related initial data/segment/job would involve the following procedures. Note that Video and images cannot be upload at the same time, and only one Video is supported for annotating task.

- POST to /task api is called to initialized the Task
- A tus protocol is started to upload all files append to write to backend with called to /task/{id}/data api. Which includes the following stages:
  - Upload started: create some directories and initialization.
  - Append files: write chunked files to its destination.
  - Upload finished: finished the data upload and api calling, which would then involve a django_rq worker to post process the task data in worker thread. Some important steps are:
    - Process input data (images, videos, archives) from either cloud storage or local files
    - Set up the task mode and storage methods
    - Calculate chunk size and Media files preprocessing (like preview frame extraction)
    - Create the related Segment, initial Job and Data models in database
    - **important:**: if local fileStorage cache is enabled, then video or images are pre-chunked and stored locally in backend to accelerate the data fetching process later. The file type of the chunked data is also configureable in setting, whether in .zip format or in .mp4 format.

#### Frame fetching procedure

The main api we use to fetch frames data for a job would be /job/{id}/data with query params: `type`, `number`/`index` and `quality`. It is used mainly in several situations:

- if type is preview, then get the preview frame for a job, use the frame number and requested quality.
- if type is chunk, then get the frames chunk for a job(segment), use the index of the chunk and the requested quality.

I will talk about the second process because they are similar in design and implementation.

The design pattern in data fetching api is a Getter/Provider design pattern. For job related data frame, `JobDataGetter`/`JobFrameProvider` is paired in use. If you follow the code flow, you will find that `JobFrameProvider` is derived from `SegmentFrameProvider` and call to `JobFrameProvider` methods will possibly transfered to call `SegmentFrameProvider`. So the main logic in get chunk frames data would be in `SegmentFrameProvider.get_chunk()` function.

The `SegmentFrameProvider` will provide different types of loaders based on the cache and storage method we set. The `_BufferChunkLoader` and the `_FileChunkLoader`.

Like we introduced in the last chapter, if local fileStorage cache is enabled, then media file would be chunked and stored in local files. In this case, `_FileChunkLoader` is used to just open the file with relate to requested quality (compressed or original), and send the data back.

In another case, `_BufferChunkLoader` is used to read the related media files on request, get the frames data with chunk_index in its requested quality, then send back. In this situation, `_BufferChunkLoader` would dispatch job to `VideoReader` or `Mpeg4ChunkWriter` or other readers in `media_extractors.py` to read data and get chunked smaller file. In this case, in-memory cache is used when enabled to store the returned data with built redis key to accelerate the possible future request.

### Backend audio feature implementation

After understanding the design principle and pattern, I finished my implementation of backend audio feature support in these aspects.

- The first change comes in adding two functions to `VideoReader`, since this is the class we use to read video data. One is `get_audio_chunk` which is used get the audio data of one sepecific chunk, and the other is `separate_audio_chunks` which would seperate all the audio chunks all at once.

> The VideoReader is reused in add fetching audio because I think there is no need to add a `AudioReader`

- As we note before, since the media file would be chunked and stored locally if local fileStorage cache is enabled. I add the same logic to audio data. In `_create_task` worker thread function, the audio would be chunked in local storage if enabled.
- Next comes to the design of the audio api, and its related Getter/Provider/_loaders. The audio data is fetched through get /job/{id}/data with query params chunk_index, since audio is only available for video and chunk type. And a new class `_JobAudioGetter` and  `SegmentAudioProvider` is added to align with the design pattern. And in `SegmentAudioProvider.get_chunk`, the function would call to different _loaders according to settings slimilar to what we talked in walk through:
  - Use `_FileChunkLoader` if enabled: directly return the chunked audio file.
  - Use `_BufferChunkLoader`, dispatch read job to `VideoReader` to get audio chunk.

I would also like to talk about the introduce of `_JobAudioGetter` since it not derived from `_DataGetter`, which serves as basic class for `_JobDataGetter`. Because in backend's original design, `_DataGetter` has a `_get_frame_provider` to be implemented, however in audio we don't have such frame data but chunk data. So instead I just create a new Class with no basic Class.


## Frontend

### Frontend walk through

The frontend use a stack of React and Redux, and the main container and component we would focus on is `src/containter/tap-bar.ts` and `src/component/top-bar.ts`. The playing frame logic is:

- Click play button on the top-bar
- Container detect a state change and check if playing conditions satisfied
- If playing satisfied then check if still have next frame to be played
- If yes, then frameNumber is changed to next frame which would result in another state change
- So the loop continues untile no more frame to play or playing condition not satisfied

When frameNumber changes, getFrame would go to check if the chunk is already fetched or need to call backend to get chunk frame data. Then update frame data in the frontend state to rerender video frame.

Also to ensure the frame played at the given frameSpeed, a calculated delay is updated in state to be awaite in `handlePlayIfNecessary` before change to next frame.

### Frontend audio feature implementaion

#### 1. Wrap the api call to backend

Actually we only have one api added, the `/api/jobs/{jid}/audio` with queryparam `index`. This is wrapped in `server-proxy.ts` file's `getAudioData()` function.

Also we need to add a function property `audio` in `Session.frames` which would be derived by `Job` or `Task`, but would only be called by `Job` instance now to get the audio of a chunk. It's definition and implementation (a wrap of `getAudioData`) is in `session.ts` and `session-implementation.ts`.

> Althrough define audio property in `Session.frames` would seem strange cause frame don't have audio but chunk have. Here my consideration is that `Session.frames` would hold all the data fetching logic related to frame, so audio data can be resonable to be fetched here. It's just a api call wrap after all.

So at this point, we can get the audio data of a chunk with call to `job.frames.audio(chunk_index)`.

#### 2. Frontend audio state management

We need to add property to represent audio state in the frontend. Since this project uses reducer and action from redux to management states, so apart froom adding states we also need to add action that would update the states.

The related actions to update the states are defined in `AnnotationActionTypes` from `cvat-ui/src/actions/annotation-actions.ts`.

The audio states is added inside of `AnnotationState.player.audio` in `cvat-ui/src/reducers/annotation-reducer.ts` since audio is played along with frame in annotation page. The detailed update logic align to actions are defined here too.

```
audio: {
    muted: boolean;
    fetching: boolean;
    chunks: Record<number, ArrayBuffer>;
    activeRequests: Map<number, Promise<void>>;
    currentChunk: number | null;
};
```

This states will be used to control the frontend audio player behavior.

#### 3. Get audio data

There are several places where getAudioData will be called.

- When open the job page, the first chunk of audio data will be loaded at the same time.

> At this moment, the `frameSpeed` would be calculated by dividing chunkSize with the duration of first audio chunk. (Actually the real fps of the backend video).

- When keep playing frame, the next chunk of audio will be loaded. Similar to the logic of prefetching the next chunk of video frames.
- When frameNumber is changed and the related audio chunk is not cached in frontend state, then load the audio chunk where the frame belongs to.

#### 4. Play sync with frame

I have tried the following two approches:

1. Play the audio with respect to each frame in the `handlePlayIfNecessary` function. Which means, at each frame the only a small part of audio chunk would be played.

2. Play the audio whenever playing state changed from false to true (staring from the frame start point), and stop playing whenever muted or stoped playing. Since we make sure the `FrameSpeed` is set to the fps of the video, the audio and frames should roughly be played synchronously.

Each of these two function has its advantage and downside. For the first approch, audio and video is synchronous all the time, but audio play would be more discrete. For the second approch, the synchrony in each chunk cannot be ensured, though with more audio playing premotion.

#### 5. Add mute button

This would be more straight forward since you just add a state called muted and is updated by clicking on the button. And the state change would cause the audioSource to disappear.