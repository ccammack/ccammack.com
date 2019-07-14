---
title: "Create Wrapper Scripts for youtube-dl"
date: 2018-06-02T23:45:37-07:00
tags: ["Batch", "FFmpeg", "Windows", "YouTube-dl"]
---

The excellent command-line video downloader [youtube-dl](https://rg3.github.io/youtube-dl/) can be made even more useful by wrapping it with a couple of batch scripts to handle the most common cases.

<!--more-->

#### Installation

1. Download and install the [Microsoft Visual C++ 2010 Redistributable Package (x86)](https://www.microsoft.com/en-US/download/details.aspx?id=5555).

1. Open the Start menu, type **cmd** to filter and select **cmd** to open a console window. Create a new folder **C:\Programs\capture** and download **youtube-dl**.

	{{< highlight bat >}}
C:\Windows\System32> mkdir C:\Programs\capture
C:\Windows\System32> cd C:\Programs\capture
C:\Programs\capture> explorer https://yt-dl.org/latest/youtube-dl.exe
C:\Programs\capture> mv %USERPROFILE%\Downloads\youtube-dl.exe .
C:\Programs\capture> start C:\Programs\capture
{{< /highlight >}}

1. Download the [static Windows build of FFmpeg](https://ffmpeg.zeranoe.com/builds/) and open the downloaded file.
Drill down the folder tree to find the **bin** directory, then copy **ffmpeg.exe** and **ffprobe.exe** to **C:\Programs\capture**.
{{< figure src="capture-folder-in-windows-explorer.png" alt="Windows explorer window showing the folder C:\Programs\capture.">}}

#### Capture MP4 Video

1. Use **Notepad** to create a new batch file that will capture video.
{{< highlight bat >}}
C:\Programs\capture> notepad capture-video.bat
{{< /highlight >}}

1. Copy and paste the following code into the new file, then save and close **Notepad**.
{{< highlight bat >}}
@echo off
setlocal

REM usage: capture-video.bat "Official Rick Astley" "https://www.youtube.com/watch?v=dQw4w9WgXcQ"

:update
%~dp0\youtube-dl.exe --update | more

:capture
set output=%~dp0\capture.mp4\"%~1"
mkdir %output% 2>nul
pushd %output%
%~dp0\youtube-dl.exe --format mp4 --merge-output-format mp4 --recode-video mp4 --prefer-ffmpeg --ffmpeg-location %~dp0 "%~2"
popd

:done
endlocal
exit /B %ERRORLEVEL%
{{< /highlight >}}

1. Run the **capture-video** script from the command line, providing a **"&lt;name&gt;"** and **"&lt;URL&gt;"** to capture, each surrounded by double-quotes.
{{< highlight bat >}}
C:\Programs\capture> capture-video.bat "Official Rick Astley" "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
youtube-dl is up-to-date (2018.06.02)
[youtube] dQw4w9WgXcQ: Downloading webpage
[youtube] dQw4w9WgXcQ: Downloading video info webpage
[youtube] dQw4w9WgXcQ: Extracting video information
[download] Destination: Rick Astley - Never Gonna Give You Up (Video)-dQw4w9WgXcQ.mp4
[download] 100% of 41.68MiB in 00:10
[ffmpeg] Not converting video file Rick Astley - Never Gonna Give You Up (Video)-dQw4w9WgXcQ.mp4 - already is in target format mp4
{{< /highlight >}}

1. Find the downloaded MP4 video file in **C:\Programs\capture\capture.mp4**.
{{< figure src="official-rick-astley-mp4.png" alt="Windows explorer window showing the downloaded file.">}}

#### Capture MP3 Audio

1. Use **Notepad** to create a new batch file that will capture audio.
{{< highlight bat >}}
C:\Programs\capture> notepad capture-audio.bat
{{< /highlight >}}

1. Copy and paste this code into the new file, then save and close **Notepad**.
{{< highlight bat >}}
@echo off
setlocal

REM usage: capture-audio.bat "Official Rick Astley" "https://www.youtube.com/watch?v=dQw4w9WgXcQ"

:update
%~dp0\youtube-dl.exe --update | more

:capture
set output=%~dp0\capture.mp3\"%~1"
mkdir %output% 2>nul
pushd %output%
%~dp0\youtube-dl.exe -x --audio-format mp3 --audio-quality 0 --prefer-ffmpeg --ffmpeg-location %~dp0 "%~2"
popd

:done
endlocal
exit /B %ERRORLEVEL%
{{< /highlight >}}

1. Run the **capture-audio** script from the command line, providing a **"&lt;name&gt;"** and **"&lt;URL&gt;"** to capture, each surrounded by double-quotes.
{{< highlight bat >}}
C:\Programs\capture> capture-audio.bat "Official Rick Astley" "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
youtube-dl is up-to-date (2018.06.02)
[youtube] dQw4w9WgXcQ: Downloading webpage
[youtube] dQw4w9WgXcQ: Downloading video info webpage
[download] Destination: Rick Astley - Never Gonna Give You Up (Video)-dQw4w9WgXcQ.webm
[download] 100% of 3.28MiB in 00:00
[ffmpeg] Destination: Rick Astley - Never Gonna Give You Up (Video)-dQw4w9WgXcQ.mp3
Deleting original file Rick Astley - Never Gonna Give You Up (Video)-dQw4w9WgXcQ.webm (pass -k to keep)
{{< /highlight >}}

1. Find the downloaded MP3 audio file in **C:\Programs\capture\capture.mp3**.
{{< figure src="official-rick-astley-mp3.png" alt="Windows explorer window showing the downloaded file.">}}
