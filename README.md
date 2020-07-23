
# VOA V3 Migration Documentation

## API Flow

1. `createTDOWithAsset` - Create a TDO with an audio, video, or text asset attached.
2. `createJob` - Create a cognition job.
3. `checkJobStatus` - Check the status of a job. (optional)
4. `retrieveEngineOutput` - Retrieve the output of a completed job.


## Migrating to V3

### Update Request Header

V3 allows application and user level tracking for API calls.

Setting a header "x-veritone-application":"org:orgGuid" allows Veritone to monitor API usage and provides better tracking for debugging purposes.  

**VOA orgGuid: "17353"**

Example: `"x-veritone-application": "org:17353"`

It isn't strictly necessary, but it does help the team diagnose future issues if they arise.


Applications **MUST**:

Set `x-veritone-application` header on all API requests. This can be retrieved from the initial url from switch-app.  See below for more information

All SSO Tokens in use must have a valid user_id and application_id.

## User Tracking

To enable user tracking, please set the following:

Token                  | To Enable                                                                                     | Comments 
---------------------- | --------------------------------------------------------------------------------------------- | -------- 
Session                | This is tracked by default                                                                    | N/A
JWT Tokens             |                                                                                               | This is typically used by engines
SSO Token              | Set JSON field `user_id` inside the JSONB<br/> column `json` on public.sso.sso_token          | This is typically for ingestion
Header                 | Set header: `x-veritone-user: UserID`                                                         | API will be tracked as the user

## Application Tracking

Token                  | To Enable                                                                                     | Comments 
---------------------- | --------------------------------------------------------------------------------------------- | -------- 
Session                | Set header: `x-veritone-application:` to "APP_ID"                                             | N/A
JWT Tokens             | Set header: `x-veritone-application:` to "APP_ID"                                             | This is typically used by engines
SSO Token              | Please set the `application_id` column on `sso_token` or set the header.                      | This is typically for ingestion
Header                 | Set header: `x-veritone-application:` to "APP_ID"                                             | 

## Example V3 API Calls

### `createTDOWithAsset`

Use the TDO ID and signedUri returned from this query in the next step.

```
mutation createTDOWithAsset {
  createTDOWithAsset(input: {
    startDateTime: 1587159404, 
    updateStopDateTimeFromAsset: true, 
    contentType: "video/mp4", 
    assetType: "media", 
    uri: "media URL"
  }) {
      id
      status
      assets {
        records {
        id
        assetType
        contentType
        signedUri
}}}}
```

### `createJob`

In V3 the createJob mutations have more definition than the previous V2 and Iron frameworks in order to make them more flexible. 

Once you have your template there are a few things you will need for each job.

- `tdoId` Use the id provided in the response of createTDOWithAsset .

- `sourceUrl` Use the signedUri provided in the response of createTDOWithAsset or a custom url.

- `engineId` Provide the id of your desired cognition engine.  Example uses an English transcription engine.

## Transcription

VOA Transcription engineId's and payload types ( Speechmatics uses different engineId's instead of payloads )

 Engine Name             | engineId                             | payload name
 ----------------------- | ------------------------------------ | --------------------------
 Microsoft Transcription | 1fe73773-edcb-4610-9c6e-cbe39eecec30 | "language"
 Google Transcription    | f99d363b-d20a-4498-b3cc-840b79ee7255 | "languageCode"
 Speechmatics English    | c0e55cde-340b-44d7-bb42-2e0d65e98255 | n/a
 Speechmatics French     | ac20a648-5ed4-486c-a31d-d553f4209e32 | n/a
 Speechmatics Russian    | e02e7daa-5920-4119-bcc4-e0470ae12680 | n/a
 Speechmatics Spanish    | 71ab1ba9-e0b8-4215-b4f9-0fc1a1d2b255 | n/a
 Speechmatics Arabic     | 52bb5731-7365-4179-b520-1bd9fdd37f62 | n/a
 Speechmatics Korean     | 718bce7f-cddb-44f1-b838-dfb9b25b8485 | n/a
 
 #### Example engine payload
 
 ```
 payload: {
  "language": "en-US"
 }
 ```
 
 ### `launchSingleEngineJob`
 
 With V3 there are still simple `createJob` style mutations from V2 but they are now called `launchSingleEngineJob`
 
 Use `launchSingleEngineJob` where supported, or use one of the template DAGs provided.
 
 Here is an example single engine job using microsoft transcription that creates a tdo for you:
 
 ```
 mutation microsoftTranscription{
  launchSingleEngineJob(input: {
    targetId: "tdo_id"
    engineId: "1fe73773-edcb-4610-9c6e-cbe39eecec30"
    fields: [
      {fieldName: "priority", fieldValue: "-10"},
      {fieldName: "language", fieldValue: "en-US"},
      {fieldName: "clusterId", fieldValue: "rt-1cdc1d6d-a500-467a-bc46-d3c5bf3d6901"}
    ]
  }) {
    id
    targetId
  }
}
```

To run this engine against a media url, provide an `uploadUrl` field instead of a `targetId`

You can also use a DAG template to create a job, these templates are verbose but give you more control.

### `audioCognition` template

```
mutation createCognitionJob {
    createJob(input: {
      targetId: "tdo_id"
      clusterId :"rt-1cdc1d6d-a500-467a-bc46-d3c5bf3d6901"
      tasks: [
        {
          # webstream adapter
          engineId: "9e611ad7-2d3b-48f6-a51b-0a1ba40fe255"
          payload: { url: "media_url" } # add the sourceUrl or signedUri from previous steps
          ioFolders: [
            { referenceId: "wsaOutputFolder", mode: stream, type: output }
          ]
          executionPreferences: { priority:-10 }
        }
        {
          # Ingester engine to break audio into 20 second chunks
          engineId: "8bdb0e3b-ff28-4f6e-a3ba-887bd06e6440"
          payload:{
            ffmpegTemplate: "audio"
            customFFMPEGProperties: { chunkSizeInSeconds: "20" }
          }
          ioFolders: [
            { referenceId: "siInputFolder", mode: stream, type: input }
            { referenceId: "siOutputFolder", mode: chunk, type: output }
          ]
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-10 }
        }
        {
          # English transcription with optional payload field
          engineId: "c0e55cde-340b-44d7-bb42-2e0d65e98255"
          payload: {}
          ioFolders: [
            { referenceId: "engineInputFolder", mode: chunk, type: input }
            { referenceId: "engineOutputFolder", mode: chunk, type: output }
          ]
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-10 }
        }
        {
          # Output writer
          engineId: "8eccf9cc-6b6d-4d7d-8cb3-7ebf4950c5f3"
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-10 }
          ioFolders: [
            { referenceId: "owInputFolder", mode: chunk, type: input }
          ]
        }
      ]
      routes: [
      { 
          # adapter --> chunker
          parentIoFolderReferenceId: "wsaOutputFolder"
          childIoFolderReferenceId: "siInputFolder"
          options: {}
        }
        { 
          # chunker --> engine
          parentIoFolderReferenceId: "siOutputFolder"
          childIoFolderReferenceId: "engineInputFolder"
          options: {}
        }
        { 
          # engine --> output writer
          parentIoFolderReferenceId: "engineOutputFolder"
          childIoFolderReferenceId: "owInputFolder"
          options: {}
        }
      ]
    }){
      id
}}
```

To use a different transcription engine, change the engineId and add a payload field (if necessary).

## Speaker Separation

Speaker separation should be run against a tdo with a transcription asset.

 Engine Name             | engineId                             | payload
 ----------------------- | ------------------------------------ | --------------------------
 Speechmatics Speaker Separation | 06c3f1d7-7424-407b-a3b5-6ef61154fc0b | n/a
 
### `speakerSeparation` using `launchSingleEngineJob`

```
mutation speakerSeparation {
  launchSingleEngineJob(input: {
    targetId: "tdo_id
    engineId: "06c3f1d7-7424-407b-a3b5-6ef61154fc0b"
    fields: [
      {fieldName: "priority", fieldValue: "-10"},
      {fieldName: "clusterId", fieldValue: "rt-1cdc1d6d-a500-467a-bc46-d3c5bf3d6901"},
    ]
  }) {
    id
    targetId
  }
}
```

### `speakerSeparation` template

```
mutation createCognitionJob {
    createJob(input: {
      targetId: 1110708367
      clusterId :"rt-1cdc1d6d-a500-467a-bc46-d3c5bf3d6901"
      tasks: [
        {
          # webstream adapter
          engineId: "9e611ad7-2d3b-48f6-a51b-0a1ba40fe255"
          payload: { url: "https://holdforfisher.s3.amazonaws.com/KarlMeltzer4.mp4" } # add the sourceUrl or signedUri from previous steps
          ioFolders: [
            { referenceId: "wsaOutputFolder", mode: stream, type: output }
          ]
          executionPreferences: { priority:-10 }
        }
        {
          # Ingester engine to break audio into 20 second chunks
          engineId: "8bdb0e3b-ff28-4f6e-a3ba-887bd06e6440"
          payload:{
            ffmpegTemplate: "audio"
            customFFMPEGProperties: { chunkSizeInSeconds: "20" }
          }
          ioFolders: [
            { referenceId: "siInputFolder", mode: stream, type: input }
            { referenceId: "siOutputFolder", mode: chunk, type: output }
          ]
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-20 }
        }
        { # Speaker Separation
          engineId: "06c3f1d7-7424-407b-a3b5-6ef61154fc0b"
          ioFolders: [
            { referenceId: "engineInputFolder", mode: chunk, type: input }
            { referenceId: "engineOutputFolder", mode: chunk, type: output }
          ]
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-20 }
        }
        {
          # Output writer
          engineId: "8eccf9cc-6b6d-4d7d-8cb3-7ebf4950c5f3"
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-20 }
          ioFolders: [
            { referenceId: "owInputFolder", mode: chunk, type: input }
          ]
        }
      ]
      routes: [
        { 
          # adapter --> chunker
          parentIoFolderReferenceId: "wsaOutputFolder"
          childIoFolderReferenceId: "siInputFolder"
          options: {}
        }
        { 
          # chunker --> engine
          parentIoFolderReferenceId: "siOutputFolder"
          childIoFolderReferenceId: "engineInputFolder"
          options: {}
        }
        { 
          # engine --> output writer
          parentIoFolderReferenceId: "engineOutputFolder"
          childIoFolderReferenceId: "owInputFolder"
          options: {}
        }
      ]
    }){
      id
}}
```
# Translation is WIP

Section will be updated as soon as the engines are finished.

### `checkJobStatus` 

You can poll for job status updates or use `notificationUris` detailed below.

```
query queryJobStatus {
  job(id:"JOB_ID") {
    id
    status
    targetId
    tasks{
      records{
        id
        status
        engine{
          id
          name
        }
        # taskOutput - include this field for a more verbose output
      }
    }
  }
}
```

### `retrieveJobOutput`

Include the TDO ID and Engine ID to retrieve transcription output in JSON format.

The example uses english transcription, change engine id for other use cases.

```
query getEngineOutput {
  engineResults(tdoId: "TDO_ID",
    engineIds: ["c0e55cde-340b-44d7-bb42-2e0d65e98255"]) { # Retrieves results from english transcription engine
    records {
      tdoId
      engineId
      startOffsetMs
      stopOffsetMs
      jsondata
      assetId
      userEdited
}}}
```

### `notificationUris`

`notificationUris` allow you to link to a custom endpoint(s).  This removes the need to poll for job completion, you will instead be notified when the job completes at the endpoint provided.

You can use the `notificationUri` field in `launchSingleEngineJob` to be notified when the engine has completed.

```
mutation singleEngineJob{
  launchSingleEngineJob(input: {
    engineId:"c0e55cde-340b-44d7-bb42-2e0d65e98255", # English transcription engine
    targetId: "TDO_ID
    fields: [
      { fieldName:"priority", fieldValue:"-10" },
      { fieldName: "notificationUri", fieldValue: "https://example.net/dump"}
    ]
  }) {
    id
}}
```

You can also set notifications at the task level for more updates on job status.

This example notifies you when the output is completed:

```
{
  # Output writer
  engineId: "8eccf9cc-6b6d-4d7d-8cb3-7ebf4950c5f3"
  executionPreferences: { parentCompleteBeforeStarting: true, priority:-10 }
  ioFolders: [
    { referenceId: "owInputFolder", mode: chunk, type: input }
  ]
  notificationUris: ["https://example.net/dump"]
}
```

This example notifies you as each task is completed:

```
mutation createCognitionJob {
    createJob(input: {
      targetId: "TDO ID"
      notificationUris: [ "https://example.net/dump" ] # This endpoint will be set for each task
      clusterId :"rt-1cdc1d6d-a500-467a-bc46-d3c5bf3d6901"
      tasks: [
        {
          # webstream adapter
          engineId: "9e611ad7-2d3b-48f6-a51b-0a1ba40fe255"
          payload: { url: "URL" } # add the sourceUrl or signedUri from previous steps
          ioFolders: [
            { referenceId: "wsaOutputFolder", mode: stream, type: output }
          ]
          executionPreferences: { priority:-10 }
        }
        {
          # Ingester engine to break audio into 20 second chunks
          engineId: "8bdb0e3b-ff28-4f6e-a3ba-887bd06e6440"
          payload:{
            ffmpegTemplate: "audio"
            customFFMPEGProperties: { chunkSizeInSeconds: "20" }
          }
          ioFolders: [
            { referenceId: "siInputFolder", mode: stream, type: input }
            { referenceId: "siOutputFolder", mode: chunk, type: output }
          ]
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-10 }
        }
        {
          # English transcription with optional payload field
          engineId: "c0e55cde-340b-44d7-bb42-2e0d65e98255"
          payload: {}
          ioFolders: [
            { referenceId: "engineInputFolder", mode: chunk, type: input }
            { referenceId: "engineOutputFolder", mode: chunk, type: output }
          ]
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-10 }
        }
        {
          # Output writer
          engineId: "8eccf9cc-6b6d-4d7d-8cb3-7ebf4950c5f3"
          executionPreferences: { parentCompleteBeforeStarting: true, priority:-10 }
          ioFolders: [
            { referenceId: "owInputFolder", mode: chunk, type: input }
          ]
        }
      ]
      routes: [
      { 
          # adapter --> chunker
          parentIoFolderReferenceId: "wsaOutputFolder"
          childIoFolderReferenceId: "siInputFolder"
          options: {}
        }
        { 
          # chunker --> engine
          parentIoFolderReferenceId: "siOutputFolder"
          childIoFolderReferenceId: "engineInputFolder"
          options: {}
        }
        { 
          # engine --> output writer
          parentIoFolderReferenceId: "engineOutputFolder"
          childIoFolderReferenceId: "owInputFolder"
          options: {}
        }
      ]
    }){
      id
}}
```
