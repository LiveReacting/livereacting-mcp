---
name: schedule-recorded-broadcast
description: Schedule one pre-recorded video to go live on YouTube, Facebook, Twitch or a custom RTMP destination at a set date and time with LiveReacting, like a premiere or a webinar replay. Use when the user wants a recorded video to air live later, at a specific time.
---

# Schedule a recorded broadcast

Air one recorded video as a live stream at a time the user chooses. The stream starts and ends
by itself. Confirm the time with the user before scheduling anything.

## 1. Confirm the time

1. Get the date, the time and the user's time zone. If the time zone is unclear, ask.
2. Convert it to an ISO 8601 time in UTC.
3. The start must be at least 3 minutes and at most 28 days in the future. If it is not, ask
   for another time.
4. Repeat the time back to the user in their time zone and in UTC.

## 2. Get the video ready

1. Call `livereacting_list_media` with `type: "video"` to find the video. Follow
   `nextOffset` before treating it as missing.
2. If it is not in the library:
   - Google Drive, Dropbox, or one video from the user's own YouTube channel:
     `livereacting_import_media`.
   - A direct link to a video file: `livereacting_import_from_url`.
3. Poll `livereacting_get_import_status` no more than every 10 seconds. Read `usable`, not only
   `done`. Stop and report an import error when `status` is `error`. Otherwise, when
   `usable` is false, follow `nextStep`.
4. Read the file with `livereacting_get_media_file`, including a file already in the
   library. If it needs encoding, call `livereacting_encode_media` once. If it is
   already queued or encoding, wait for that job instead.
5. Poll `livereacting_get_encode_status` no more than every 15 seconds. After each check,
   read the selected file again with `livereacting_get_media_file`. While `issues.errors`
   includes `encoding_in_progress`, keep waiting. A file missing from one queue page is
   not proof that encoding finished or succeeded.
6. Once encoding is no longer active, require `issues.errors` to be empty. If errors
   remain, stop and report them. Do not automatically encode the same file again.
7. Call `livereacting_get_media_file` with `includeMetadata: true` and note `durationSec`.
   Require a positive duration; if it is missing, stop rather than inventing a length.

## 3. Build the broadcast

1. Call `livereacting_list_destinations` and confirm the destination with the user. If it is
   missing, the user must connect it once in the LiveReacting Studio.
2. Call `livereacting_create_project` with the destination ids and a fixed duration:
   `duration: { manual: false, planned: <durationSec rounded up, at least 30> }`. The stream then
   ends when the video ends. Do not add extra time: the stream keeps running for the whole planned
   duration, after the video has finished.
3. Call `livereacting_create_scene_with_playlist` with the one file id and `loop: false`, so the
   video plays once.

## 4. Schedule it

1. Call `livereacting_get_project` with `includeScheduledLives: true`. If the project already has
   a schedule, ask whether the user wants to replace it. Do not cancel it yet.
2. Recheck the requested start against the current UTC time. It must still be at least
   3 minutes and at most 28 days ahead. Preparation may have taken several minutes. If it is
   outside that range, stop and ask for another time. Never move it silently.
3. If an existing schedule needs replacement and the user authorized it, call
   `livereacting_cancel_schedules`. Recheck the time range again after cancellation.
   If the time is now outside the range, stop and ask for another time before scheduling.
4. Immediately after the time check, call `livereacting_schedule_stream` with the created
   `projectId` and `dates: [<the UTC time>]`. If it rejects the date as too soon, ask for another
   time instead of retrying the same date or choosing a new time yourself.
5. After a successful response, report the confirmed start time in the user's time zone and
   in UTC, the length, and the destination. Say that LiveReacting starts and stops it by itself,
   so the user's computer can be off.

## Good to know

- Do not call `livereacting_start_stream` on a scheduled project: it returns the existing
  schedule and starts nothing.
- To change the time, cancel the schedule with `livereacting_cancel_schedules`, then schedule
  again.
- Title and description changes made later with `livereacting_update_project` also update the
  scheduled broadcast on the platform.
