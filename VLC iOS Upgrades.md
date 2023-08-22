# VLC iOS Upgrades (Google Summer of Code 2023)

Through the Google Summer of Code program, I've had the privilege to contribute to VLC's iOS application under the mentorship of [Diogo Simao Marques](https://code.videolan.org/diogo.simao-marques) and [Felix Paul Kühne](https://code.videolan.org/fkuehne). I've worked on multiple issues and merge requests over the past 3 months, ranging from simple bug-fixes to new features. The following is a list of my merge requests and my experiences working on them. 

## Improved System Integration: Siri Support

- MR link: [Draft: Siri/Shortcut/Media Intent support](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1082) (Not merged yet)

iOS offers "Intents" as a way to integrate actions within your app with the rest of the operating system. To integrate with Siri, you must implement handling for specific Sirikit Intents. The Intents I chose to implement were INPlayMediaIntent (to play media through Siri) and INAddMediaIntent (to add the currently playing media to a playlist).

Initially I implemented these in a separate Intent Extension (to support iOS 13+), but sharing the media library between the application and extension proved to be too difficult to maintain. Hence iOS 13 support for Sirikit intents was dropped and I moved to embedding the intent support within the app itself (now supporting iOS 14+, note that VLC for iOS supports iOS 9+)

### INPlayMediaIntent

This was the coolest feature to work on, hands down. I implemented a new Intent Coordinator class that would be called by the App Delegate whenever Siri would invoke the handlerForIntent() method.

This intent required two methods: resolveMediaItems() and handle(). Unlike most intents, a confirm() was not needed (Siri bugging you with a confirmation prompt every time you wanted to play something would result in a very poor user experience)

#### resolveMediaItems()

This method is automatically called by the intent handler in my coordinator.

This method received an INPlayMediaIntent, and my goal was to return a "resolution result" out of it. This meant extracting the search query, the media type, and any other parameters from the user's search. Digging through Apple Documentation and WWDC 2019 slides, I understood how to parse Siri's query into useful parameters for VLC's Media Library Service.

I implemented helper methods to search for the right media type (a song? an album? an artist?) and to perform a library-wide search if the type was "unknown".

As a side note, Siri often returns queries with an unknown media type. Users rarely specify the type: how many of us say "Siri, play the song $SONG"? We'll say "Siri, play $SONG".

This helper method returns an optional INMediaItem. Based on the item a resolution result of supported or unsupported is passed.
#### handle()

Handle receives the INMediaItem passed from resolve() and plays the media. I retrieve it's identifier, retrieve it from the media library and play it.

Additionally, I read other properties from the user's query (such as shuffle mode, playback queue location, repeat mode, playback speed) and adjust the playback as needed. 

Lastly, if the media that was selected for playback is a video, I send a response code of .continueInApp, causing Siri to open my app in the foreground. For all other purposes (such as music playback) VLC continues to play in the background.

Here's a screenshot of it!

<img src="siri.gif" width="250"/>

### INAddMediaIntent

For practical purposes, I limited the application of this intent to only support the currently playing media as the media item, and only playlists as a viable source to add media. This is because in VLC for iOS, media is either in the library or on the network. 

This intent also required two methods: resolveMediaItems() and handle()

#### resolveMediaItems()

Unlike the resolve I wrote for the INPlayMediaIntent, I retrieved the media item from the Siri query. I also made sure the Siri query only pertained to the currently playing media. Lastly, I retrieved the media's identifier from the VLC Playback Service.

#### handle()

I retrieved the user's playlist from the Siri query, and retrieved the media item from the identifier. If both existed, I called the appendMedia() method in the VLCMLPlaylist object.


### INSearchForMediaIntent

During my final week working at GSoC, I also tried to implement support for searching for files within Siri to open up the app. As you can guess, this intent is not very popular and nor is it easy to support. Support for it has been added as a stub, but I doubt the VLC team will include it.

## Major UI Additions / Fixes

### A History page of Recently Opened Media

- MR link: [History: History page for video & audio](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1066) (Approved, not merged)

A feature targeted at power-users with big libraries, enabling you to scroll through your history in chronological order. VLC iOS already had a "history" of media the user was watching, this API just wasn't accessible in the UI. 

To design this, I had to refer back to the existing media views (such as the Album view, the Media Group view). This meant designing a new History Category View Controller that was of type MediaCategoryViewController, and a new History Model in parallel.

The History Model read the history from VLC Media Library Service's history(of:) method.  I also subscribed to the "historyChangedOfType" notification in the media library to refresh the History Page on any change in History.

Lastly, I added a button in the navigation bar for the Video and Audio Views to view the history of their respective media types.


<img src="https://code.videolan.org/videolan/vlc-ios/uploads/690a4d78e2a54b876377e73f3eac9fc2/Screen_Recording_2023-06-13_at_9.50.49_PM.gif" width="250"/>

### Select files for deletion while searching simultaneously

- MR link: [Search While Delete: Pass searchDataSource to EditController in initialization.](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1072) (Approved, not merged)

The "Edit" button is hidden when the user starts searching in the library. This MR re-enables editing/selection/deleting and searching simultanously. 

I passed the search data source to the edit controller, and created a new computed variable in the media and edit controllers to dynamically return a data source based on whether the user was currently searching something.

This flexibility between data sources and edit controllers resulted in this MR requiring more debugging and less programming. 

### Playback Controls Settings: Skip Duration Flexibility

- MR link: [Skip Duration Customizability](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1078) (Approved, not merged)

VLC iOS allows multiple forms of skips:
1. Gesture based (swiping to seek)
2. Tap based (double tapping to seek)

This MR brought more flexibility to the seek options. It enables power users to choose different seek durations for every possible seek (forward, backward, gesture, tap, etc). Meanwhile this MR hides the extra options for regular users, presenting Seeks depending on your choice: separate seek durations based on direction, or based on gesture, or 1 unified duration for every possible type of seek.

Behind the scenes, this required changes in the Player Controller to change the seek duration used as per the user boolean. It also required changes in the Setting Controller, to dynamically display different seek options as per the user boolean for seek bindings. The Settings Controller was also responsible for dynamically changing the text used for the options (1 "Seek" option vs 2 "Forward Seek" and "Backward Seek" options). 

This was done to simplify the options for those that wanted something simple, and also enable heightened customization for those that required it

<img src="https://code.videolan.org/videolan/vlc-ios/uploads/51691e77393e994845c9c7de7069b4f9/Simulator_Screen_Recording_-_iPhone_14_-_2023-07-05_at_02.33.31.gif" width="250"/>

### Upgraded Delete Alert: Display file name

- MR link: [Upgraded Delete Alert: Display file name](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1051) (Approved, not merged)

The existing Delete Alert within VLC was too generic and did not mention any details of the selected items. The user could very easily have selected the wrong file for deletion. 

My new MR passes the filename & file count to the Delete Alert. I receive both from the Collection View Controllers and the selected path index array. The new Delete Alert is more representative of the selected media. I had to attempt casting the current view's collection view to each possible VLC Collection View: a MediaCollectionViewCell, a MediaGridCollectionCell, or a MovieCollectionViewCell. I then retrieve the filename from the label of the cell at the selected index.

<img src="https://code.videolan.org/videolan/vlc-ios/uploads/dfe29a60ee7bf7f05e8c148391f488ab/Simulator_Screenshot_-_iPhone_11_-_2023-06-07_at_17.37.20.png" width="250"/>


## Minor UI Fixes

### Audio Player: Improve contrast between status icons and background

- MR link: [Audio Player: Improve contrast between status icons and background](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1058) 

### Album Collection View: Prevent navigation bar from flickering when turning transparent, status bar UI fix for album View

- MR link: [Album Collection View: Prevent navigation bar from flickering when turning transparent, status bar UI fix for album View](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1054) 

### Audio Player: Hide system volume indicator, show VLC Volume indicator on volume change

- MR link: [Audio Player: Hide system volume indicator, show VLC Volume indicator on volume change](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1070)

### Cloud View Controller: Prevent "Cloud Services" title glitch when pushing new view controller

- MR link: [Cloud View Controller: Prevent "Cloud Services" title glitch when pushing new view controller](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1069)

### Disable large titles for network & settings views

- MR link: [Disable large titles for network & settings views](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1094) (Approved, not merged)

### Fix playback-scrubbing UI for paused media

- MR link: [Draft: Playback-scrubbing fixes for paused media.
](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1099) 

### Settings disparity: Fix disparity between Settings.app and VLC defaults

- MR link: [Settings disparity: Fix disparity between Settings.app and VLC defaults](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1073)

### 2 finger pan gesture support to select media

- MR link: [2 finger pan gesture support to select media](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1080)

## Minor bugfixes

### VLCPlaybackService: Remove redundant observer for "remainingTime"

- MR link: [VLCPlaybackService: Remove redundant observer for "remainingTime"](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1093) (Approved, not merged)

Both "time" and "remainingTime" properties are observed in the VLCPlaybackService, when both change simultaneously. Removing the observer on the remainingTime property increases efficiency.

### Album Header: Play All correctly opens Audio Player

- MR link: [Album Header: Play All correctly opens Audio Player](https://code.videolan.org/videolan/vlc-ios/-/merge_requests/1083)

Made sure that setting the fullscreen mode in the VLCPlayerDisplayController wouldn't call the Video Player for audio files.

## Statistics

Total Merge Requests opened by me (excluding duplicates/closed): [15](https://code.videolan.org/videolan/vlc-ios/-/merge_requests?scope=all&state=all&author_username=aviwad) 

## Conclusion

I am grateful to Google and the entire VideoLAN team for this opportunity to learn and grow. I have learnt a lot, whether it be communication skills by sending a daily log to my mentors, or it be my iOS Development skills by learning how to bridge Swift and Objective C Code. I also learnt a great deal of UIKit by admiring and adding on to the VLC iOS codebase. Thank you.