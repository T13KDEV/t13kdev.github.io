---
title: How to record a Screen with FFMPEG
image: 
  path: https://i.ibb.co/ZXQ3Y5J/e9ccba913393.jpg
sitemap: true
categories: [Apple]
tag: [macOS]
comments: false
---

## MacOS 
### Installation

```bash
brew install ffmpeg
```
> **Important:** Homebrew installs a lot of dependencies as well.
{: .prompt-tip }

### Usage
Show available devices

```bash
ffmpeg -f avfoundation -list_devices true -i ""
```
![Show available devices](https://i.ibb.co/DQw7fks/2cf0571b6820.png)

Capture Screen 1 and Audio 0

```bash
ffmpeg -t 15 -r 30 -f avfoundation -an -i 5:0 /Users/thomas/Desktop/recording.mkv
```

### EUC Score Setting 
Capture Screen 2 60 fps., pixel format uyvy422 without Audio.

```bash
ffmpeg -t 60 -f avfoundation -pixel_format uyvy422 -i "3:none" /Users/thomas/Desktop/SL2-Author_001.mkv
```

| Parameter | Description               |
| :-------: | :------------------------ |
|    -t     | Recording time in seconds |
|    -r     | Frame rate 30 fps         |

[1]: http://johnriselvato.com/ffmpeg-how-to-record-macos-screen/

