---
name: create-24-7-channel
description: Build and start a 24/7 LiveReacting live stream that loops pre-recorded videos on YouTube, Facebook, Twitch or a custom RTMP destination. Use when the user wants a continuous, always-on or looping live stream from their videos, or wants to add videos to such a channel.
---

# Create a 24/7 channel

Use the new-channel steps below only when the user requests a new channel. For changes to
an existing channel, follow 'Changing the playlist later' instead. Do not create another
project or start a stream for a playlist edit. Respect requests to prepare a channel without
starting it. Ask before going on air unless the user already authorized that action.

## 1. Choose where it streams

1. Call `livereacting_list_destinations`.
2. If the destination the user wants is missing, stop. Tell the user to connect it once in the
   LiveReacting Studio (studio.livereacting.com). Destinations cannot be added from here.
3. Confirm the destination by name with the user.

## 2. Get the videos ready

1. Call `livereacting_list_media` with `type: "video"` and find the videos the user named.
   Follow `nextOffset` before treating a video as missing.
2. For a video that is not in the library yet:
   - Google Drive, Dropbox, or one video from the user's own YouTube channel:
     `livereacting_import_media`. YouTube playlist and channel links are refused; import one
     video link at a time.
   - A direct link to a video file: `livereacting_import_from_url`.
   - Only one import runs at a time per account. Import the videos one after another.
3. Poll `livereacting_get_import_status` no more than every 10 seconds. `done` is not enough:
   read `usable`. Stop and report an import error when `status` is `error`. Otherwise,
   when `usable` is false, follow `nextStep`.
4. Read each selected file with `livereacting_get_media_file`, including files already in
   the library. If it needs encoding, call `livereacting_encode_media` once. If it is
   already queued or encoding, wait for that job instead.
5. Poll `livereacting_get_encode_status` no more than every 15 seconds. After each check,
   read the selected file again with `livereacting_get_media_file`. While `issues.errors`
   includes `encoding_in_progress`, keep waiting. A file missing from one queue page is
   not proof that encoding finished or succeeded.
6. Once encoding is no longer active, require `issues.errors` to be empty. If errors
   remain, stop and report them. Do not automatically encode the same file again.
7. Every video must pass this file-level check before building or changing a playlist.

## 3. Build the channel

1. Call `livereacting_create_project` with the destination ids, `duration: { manual: true }` and
   `autoSynch: true`. A project without a duration stops after one hour.
2. Call `livereacting_create_scene_with_playlist` with the video file ids in the order the user
   wants. The playlist loops by default.
3. Call `livereacting_validate_destinations` for the chosen destinations. If one is not valid,
   tell the user to reconnect it in the Studio before going live.

## 4. Go live

1. State the destinations and that the new stream is public. Obtain authorization to
   start unless the user already gave it. Stop here for a prepare-only request.
2. Call `livereacting_start_stream` once. Never repeat it after a timeout: call
   `livereacting_get_stream_status` instead. `PREPARING` or `LIVE` means it started.
3. Poll `livereacting_get_stream_status` no more than every 5 seconds for at most two
   minutes. Stop polling when it reports `LIVE`, `OFFLINE`, an overall error, or a
   destination error or completed destination. Report which destinations succeeded
   and which failed; overall `LIVE` does not prove that every destination works.
4. If it still reports `PREPARING` after two minutes, report that it is still starting
   and include the project and live ids for a later status check. Do not start again
   or stop the stream automatically.
5. Share a destination watch link only when its `viewingUrl` is present. If a
   destination reports `AUTH_ERROR`, the user must reconnect it in the Studio.

## Limits to tell the user up front

- A 24/7 stream needs a plan without a stream-length limit. Every plan stops the stream at its
  maximum length, even with a manual duration.
- A Facebook-only stream stops after 8 hours. Add another destination for longer runs.
- The account allows 150 requests per minute across all tools. Do not poll faster than stated.

## Changing the playlist later

1. Find the existing project with `livereacting_list_projects`, following `nextOffset`
   as needed. Use `livereacting_get_scenes` to find its playlist layer. If more than
   one project or playlist matches, ask which one the user wants to change.
2. For added videos, follow section 2 to import and check them. Then return here; do
   not run the new-channel build or start steps.
3. Read the whole playlist with `livereacting_get_layer`, using the existing `projectId`
   and `layerId`. Follow `nextOffset` until it is null.
4. Tell the user before writing: replacing the playlist restarts playback from the
   first video. Preserve the existing order and settings except for the requested edit.
5. Apply the change to the complete list and send it to `livereacting_set_playlist` with
   the existing `projectId` and `layerId`. Sending only the new video deletes the others.
   After a timeout, read the playlist instead of repeating the write.
6. Report whether the response says the edit reached the running stream. If it did not,
   explain that the saved playlist changed; do not claim that viewers see the edit.
