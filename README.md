# YouTube Cloud Delivery - FlashData

Submit an authorized media delivery to your own cloud storage, or recover its status and JSON delivery manifest using the saved job ID.

This action calls the `youtube_download` FlashData API and writes files for subsequent GitHub Actions steps. It does not invoke a language model. Paid asynchronous job. Requires customer storage. Save the receipt; never submit again to poll. Result JSON expires after 24 hours; media retention depends on storage.

## Quick start

1. [Create a FlashData account](https://flashdata.dev/signup), obtain an API Key, and fund the required Credits. [API setup and examples](https://flashdata.dev/docs/sources/youtube_download).
2. Add FLASHDATA_API_KEY as a repository or environment Actions secret. Do not use a plaintext workflow input for the key.
3. Add this manually triggered workflow and review its query before running it. Pin actions to a reviewed full commit SHA for production; the example uses a version tag for readability.

```yaml
name: YouTube Cloud Delivery - FlashData
on:
  workflow_dispatch:
    inputs:
      job_id:
        description: Previously saved FlashData download job ID
        required: true
        type: string
permissions:
  contents: read
jobs:
  collect:
    runs-on: ubuntu-latest
    steps:
      - name: Submit an authorized media delivery to your own cloud storage, or recover
          its status and JSON delivery manifest using the saved job ID.
        id: data
        uses: rocosen/flashdata-youtube-download-action@v0.1.1
        with:
          api-key: ${{ secrets.FLASHDATA_API_KEY }}
          operation: result
          job-id: ${{ inputs.job_id }}
      - name: Save data for the next pipeline stage
        uses: actions/upload-artifact@v4
        with:
          name: youtube-download
          path: ${{ steps.data.outputs.artifact-directory }}
          retention-days: 7
```

The action is bundled for Node 24; no npm install or Python setup is required in your consuming workflow. Use an up-to-date GitHub runner. No schedule is enabled by this example.

## Inputs

| Input | Required | Meaning |
| --- | --- | --- |
| `api-key` | Yes | FlashData API Key. Pass secrets.FLASHDATA_API_KEY; requests use paid Credits. |
| `video-id` | No | 11-character YouTube video ID. |
| `options-json` | No | Optional JSON object with only source-specific options listed in the README. Never include credentials here. |
| `operation` | No | status or result reads a saved job; submit creates one paid job. Default: `status`. |
| `job-id` | No | Saved job ID for status/result. Do not pass submission inputs when resuming. |
| `allow-submit` | No | Must be exactly true to submit a new paid storage write. Default: `false`. |
| `storage-json` | No | For submit only: customer storage JSON from a GitHub Actions secret. |

### Source options

`download_type` (audio_video, video, audio), `video_quality` (best, worst, or <=/==/>= plus 144, 360, 480, 720, 1080, 1440, 2160, 4320), `fps` (>=/<=/== and 1–240), `audio_language` (default, original), `audio_quality` (best, 320, 256, 192, 128), `output_format` (video: mp4/webm/mkv; audio: m4a/mp3/opus), `yt_dlp_format`, `upload_metadata` (boolean), `start_at`/`end_at` (HH:MM:SS, end after start, at most 12 hours). Default audio_video/<=720/mp4; audio defaults to m4a/best. Advanced yt_dlp_format overrides video quality, fps and audio language selection. Read operations do not accept options-json.

Unknown fields, incorrect types and invalid ranges fail before an API request. 

## Outputs

| Output | Meaning |
| --- | --- |
| `artifact-directory` | Absolute directory with response.json, records.jsonl and manifest.json. |
| `results-file` | Absolute path to the original response JSON (sanitized lifecycle fields for download jobs). |
| `records-file` | Absolute path to JSONL records for downstream steps. Empty file for an empty result. |
| `result-count` | Number of exported records; not the requested result count. |
| `source` | The fixed FlashData source used by this action. |
| `job-id` | Persist this receipt and reuse it for status/result. |
| `job-status` | Current lifecycle status. A successful action does not mean the job has completed. |

Raw API fields: submit: id/status receipt; reads: job lifecycle; completed result: output JSON delivery manifest. records.jsonl exports the sanitized job receipt/status or completed delivery manifest. manifest.json records the source, operation, timestamps, exported count and missing-collection warnings. For ordinary queries it also includes the query/options and API-reported usage.

Empty results produce an empty JSONL file and result-count 0. Missing fields stay missing; response.json preserves the original result envelope. The supplied upload-artifact step stores these files for 7 days. Only use queries and datasets appropriate for the repository’s artifact access level; private signed delivery links may be present in download manifests.

## Submit once, then resume

The default operation is `status`. The example above only reads an existing job. To create a new paid delivery, explicitly use:

```yaml
- uses: rocosen/flashdata-youtube-download-action@v0.1.1
  id: submit
  with:
    api-key: ${{ secrets.FLASHDATA_API_KEY }}
    operation: submit
    allow-submit: 'true'
    video-id: YOUR_VIDEO_ID
    options-json: '{"download_type":"audio","output_format":"m4a"}'
    storage-json: ${{ secrets.FLASHDATA_STORAGE_JSON }}
```

Replace YOUR_VIDEO_ID with an 11-character video ID you may process. Save `steps.submit.outputs.job-id`; use it with `operation: status` or `result`, omitting video-id, options-json and storage-json. These read operations never create a new job and do not poll automatically. A pending/running result is not completed delivery. Result JSON is retained for 24 hours; media retention follows your storage policy.

Store customer storage configuration in the secret FLASHDATA_STORAGE_JSON. Supported storage_type values are s3, s3_compatible, gcs, azure, oss, bos, cos and tos. All require storage_url (bucket/container); object_prefix is optional. S3-like providers require access_key/secret_key; regional providers require their documented region; s3_compatible requires an HTTPS endpoint_url. GCS uses a service_account object; Azure uses account_name and a writable container sas_token. [Provider-specific setup](https://flashdata.dev/docs/sources/youtube_download). Never paste real storage credentials into workflow source.

## Cost and failure handling

The MIT action is free to install. FlashData API calls use paid Credits; GitHub runner/artifact, model (if you add one) and customer storage costs are separate. [Current pricing](https://flashdata.dev/pricing). Extended YouTube Search has a separate rate; media delivery is billed by delivered volume. Do not infer price from sample usage fields.

A timeout, HTTP error or response-processing failure may happen after a paid request was accepted. Check [FlashData](https://flashdata.dev) Jobs before rerunning; the action does not retry POST requests. HTTP 401 means check the Key; 402 means insufficient Credits; 429 means rate/quota limit. Keep any download job ID and resume with a read operation. Do not use workflow reruns as polling. Requests have a 65-second per-request timeout and a 16-MiB response limit.

## Development and validation

Run `npm ci`, `npm run check`, `npm test` and `npm run build`. Offline tests use labeled fixtures and make no production API calls. A local or CI test pass is not proof of a paid live run or Marketplace publication. Commit the bundled dist/index.cjs together with source changes.

[Source-specific documentation](https://flashdata.dev/docs/sources/youtube_download) · [Pricing](https://flashdata.dev/pricing) · [Sign up](https://flashdata.dev/signup)
