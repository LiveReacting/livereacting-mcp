# LiveReacting

Run a 24/7 live stream of pre-recorded videos, or live stream one recorded video at the time you
choose. Your old videos, lessons, podcast episodes or music play as one continuous live stream,
looped from a playlist in the order you choose. LiveReacting's cloud servers do the streaming, so
your computer can be off.

This plugin connects Claude to your LiveReacting account. You build the stream from your video
library, multistream it to YouTube, Facebook, Twitch or a custom RTMP destination, schedule it to
go live later, change the playlist while it is live, and check how it is running.

## What you need

- A LiveReacting account on a plan that includes API access.
- Your YouTube, Facebook, Twitch or RTMP destinations connected once in the
  [LiveReacting Studio](https://studio.livereacting.com). Destinations cannot be added from Claude.

## Connect

The plugin adds the LiveReacting connector, `https://mcp.livereacting.com/mcp`. On claude.ai and in
Cowork, open the plugin's Connectors tab and connect it. In Claude Code, the server loads with the
plugin. Either way, you sign in with your LiveReacting account and approve access. The plugin holds
no passwords or keys.

## Use it

Ask in your own words, for example:

- "Loop the videos from my LiveReacting library as a 24/7 live stream on my YouTube channel."
- "Live stream my recorded video from Google Drive on my Facebook page today at 8 PM."
- "Show the status of my running live streams."

Two skills guide Claude through the main jobs:

- **create-24-7-channel**: imports and checks the videos, builds a looping playlist, and starts the
  stream after you confirm. It also changes the playlist of a channel that is already running.
- **schedule-recorded-broadcast**: prepares one video and schedules it to go live at a set time.
  LiveReacting starts and stops the stream by itself.

Claude confirms the schedule time with you, asks before it starts a stream or cancels a schedule,
and tells you before it replaces a playlist.

## Data

The plugin itself runs no code and stores nothing. It contains instructions for Claude and the
address of the LiveReacting connector.

When you use it, Claude sends your requests to your LiveReacting account through
`mcp.livereacting.com`: project names, stream titles and descriptions, video links you give it,
playlist order, destinations, schedule times and files you choose to upload. LiveReacting then
fetches the videos from the links you provide (Google Drive, Dropbox, your own YouTube channel or a
direct file link) and streams to the destinations you connected. See the
[privacy policy](https://www.livereacting.com/privacy-policy) for how LiveReacting handles this data.

## Support

- Documentation: [developers.livereacting.com/mcp](https://developers.livereacting.com/mcp/)
- Support: [livereacting.com/contact](https://www.livereacting.com/contact)
- License: MIT
