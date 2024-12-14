---
title: 'Backend Video Processing App for Streaming Platform'
description: 'To be uploaded in streaming platform we need to process it to hls format video which it will unlock new features like resolution choices, video slicing, etc.'
publishedAt: '2024-12-13'
updatedAt: ''
---

I've been working on my new project recently and I found this issue with my app, the app is a music e-learning platform which use a lot of videos to teach. The issue is it will stuck forever if we have the slow speed internet which we can't change the resolution in .mp4, fortunately I found this technique that have been used by Youtube, Netflix and Twicth. The techinque called HLS (HTTP Live Streaming), which it will format the video into segments and it will has the master file that contains all locations for the resoultions and segments of the videos.

The flow for this backend app is we will upload the video through form-data format, the backend app will process the video and the result will be one master .m3u8 and a few folders depends on the resolution setting we configure which each folders will contain segment of the videos and an index m3u8 file for the master file of each resolutions, then the backend will upload the file to storage server, in my case will be [uploadthing](uploadthing.com) and because uploadthing is non folder storage type, I need to change the location for each m3u8 files to the video's link.

I will explain it furthermore with details and will provide the github repository. If some problems happen, feel free to ask me in the comment.

# Stack that I will use

For this project I will use [Nest js](https://nestjs.com/), which it will up to you to use which frameworks and languages, but for the project I need to use js frameworks because I use [uploadthing](https://uploadthing.com) that currently just support js for its API. The video processing will be using [ffmpeg](https://www.ffmpeg.org/).

# Setting new project

### Instalation

As started, we need to install nest as global from npm, you can use this command to install and create a new project

```bash
npm i -g @nestjs/cli
nest new video-processing-backend
```

If you are not really familiar with Nest js, that's okay, take your time, because at first I didn't really have a fast adaption to be able work directly with Nest js as it has a different type of syntax.

Install ffmpeg using this command or for Windows users you can download it here [https://www.ffmpeg.org](https://www.ffmpeg.org).

```powershell
# Windows
scoop install ffmpeg
```

```bash
# Macos and Linux
brew install ffmpeg
```

We need to install multer and @types/multer to be able receive file data, to install it follow these commands, I will be using npm.

```bash
npm i multer
npm i -D @types/multer
```

### Setup

For beginning setup, the thing you must do is check if you need cors or not, if you will need it, enable cors in main.ts

```ts
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { existsSync, mkdirSync } from 'fs';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  // if need cors
  app.enableCors();
  if (!existsSync(join(__dirname, 'public')))
    mkdirSync(join(__dirname, 'public'));
  await app.listen(3000);
}
bootstrap();
```

Before we dive deeper into ffmpeg and process videos, we need to setup the api to be able receive post request and we will name the path to process, you can see the code below.

```ts
import {
  Controller,
  Get,
  HttpException,
  HttpStatus,
  Post,
  UploadedFile,
  UseInterceptors,
} from '@nestjs/common';
import { AppService } from './app.service';
import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Post('process')
  @UseInterceptors(
    FileInterceptor('video', {
      storage: diskStorage({
        destination: './dist/public'
        },
      }),
    }),
  )
  async processVideo(@UploadedFile() file: Express.Multer.File) {
    try {
      this.appService.postProcess(file);
      return 'video is being proccessed';
    } catch (e) {
      throw new HttpException(
        { status: 'error', error: e },
        HttpStatus.INTERNAL_SERVER_ERROR,
      );
    }
  }
}

```

We will put the videos into public folder, where the videos will be named with random string and one thing you need to put your attention is this app need storage to store video and we can delete it after the process is done.

Don't forget to add ConfigModule to AppModule too.

```ts
// app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

# Process videos

In nest js, we better to do logical in services, so in controller we only need to return function from service, like the code above.

Let's move to app.service.ts, we can see the generated function from nest getHello(), we can ignore it and create new function called ffmpegProcess.

```ts
// app.service.ts
ffmpegProcess(file: Express.Multer.File) {
    return new Promise<void>((resolve, reject) => {
      const rootPath = __dirname + '/public/';
      const ffmpegConsole = exec;

      ffmpegConsole(
        `ffmpeg -hide_banner -re -i ${rootPath + file.filename} -map 0:v:0 -map 0:a:0 -map 0:v:0 -map 0:a:0 -map 0:v:0 -map 0:a:0 -c:v h264 -profile:v main -crf 20 -sc_threshold 0 -g 48 -keyint_min 48 -c:a aac -ar 48000 -filter:v:0 scale=w=850:h=480:force_original_aspect_ratio=decrease -maxrate:v:0 1400k -bufsize:v:0 2100k -b:a:0 128k -filter:v:1 scale=w=1280:h=720:force_original_aspect_ratio=decrease -maxrate:v:1 2996k -bufsize:v:1 4200k -b:a:1 128k -filter:v:2 scale=w=1920:h=1080:force_original_aspect_ratio=decrease -maxrate:v:2 5350k -bufsize:v:2 7500k -b:a:2 192k -var_stream_map "v:0,a:0,name:480p v:1,a:1,name:720p v:2,a:2,name:1080p" -master_pl_name master.m3u8 -f hls -hls_time 10 -hls_playlist_type vod -hls_list_size 0 -hls_segment_filename "${rootPath}v%v/segment%d.ts" ${rootPath}v%v/index.m3u8`,
        (err, _, __) => {
          if (err) {
            reject(new Error(err.message));

            return;
          }

          rmSync(rootPath + file.filename);

          resolve();
        },
      );
    });
}
```

As you can see, it will use the terminal command for ffmpeg, which you can customize it to suit your needs, in this case I will use only three resolutions which 1080p, 720p and 480p, the output will divide the files into three folders with resolution as the title. Don't forget to remove the file we store after process it so it won't become trash that filling our storage.

To explain further, the settings are ```-map -0:v:0 -map 0:a:0``` for each resolution, so if you have four resolutions you will need for of them, then we continue to to ```scale=w=1280:h=720``` which the resulution width and height also the settings you can change within the resolution are maxrate, bufsize, the audio bitrate and the last thing for duration per segment with ```hls_time```. For the rest, you can follow my settings.

We move on to the return for controller, it's just a simple try catch to prevent the app crash if the process fails, also remove all the files.

```ts
// app.service.ts
async postProcess(file: Express.Multer.File): Promise<string> {
    try {
      await this.ffmpegProcess(file);

      return 'video process successfully';
    } catch (e) {
      const rootPath = __dirname + '/public/';

      rmSync(rootPath + 'v480p', { recursive: true, force: true });
      rmSync(rootPath + 'v720p', { recursive: true, force: true });
      rmSync(rootPath + 'v1080p', { recursive: true, force: true });

      if (existsSync(rootPath + 'master.m3u8'))
        rmSync(rootPath + 'master.m3u8');

      if (existsSync(rootPath + file.filename))
        rmSync(rootPath + file.filename);

      return 'video process failed';
    }
}
```

And for the full code will be like this.

```ts
/* eslint-disable @typescript-eslint/no-unused-vars */
import { Injectable } from '@nestjs/common';
import { exec } from 'child_process';
import { existsSync, rmSync } from 'fs';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }

  async postProcess(file: Express.Multer.File): Promise<string> {
    try {
      await this.ffmpegProcess(file);

      return 'video process successfully';
    } catch (e) {
      const rootPath = __dirname + '/public/';

      rmSync(rootPath + 'v480p', { recursive: true, force: true });
      rmSync(rootPath + 'v720p', { recursive: true, force: true });
      rmSync(rootPath + 'v1080p', { recursive: true, force: true });

      if (existsSync(rootPath + 'master.m3u8'))
        rmSync(rootPath + 'master.m3u8');

      if (existsSync(rootPath + file.filename))
        rmSync(rootPath + file.filename);

      return 'video process failed';
    }
  }

  ffmpegProcess(file: Express.Multer.File) {
    return new Promise<void>((resolve, reject) => {
      const rootPath = __dirname + '/public/';
      const ffmpegConsole = exec;

      ffmpegConsole(
        `ffmpeg -hide_banner -re -i ${rootPath + file.filename} -map 0:v:0 -map 0:a:0 -map 0:v:0 -map 0:a:0 -map 0:v:0 -map 0:a:0 -c:v h264 -profile:v main -crf 20 -sc_threshold 0 -g 48 -keyint_min 48 -c:a aac -ar 48000 -filter:v:0 scale=w=850:h=480:force_original_aspect_ratio=decrease -maxrate:v:0 1400k -bufsize:v:0 2100k -b:a:0 128k -filter:v:1 scale=w=1280:h=720:force_original_aspect_ratio=decrease -maxrate:v:1 2996k -bufsize:v:1 4200k -b:a:1 128k -filter:v:2 scale=w=1920:h=1080:force_original_aspect_ratio=decrease -maxrate:v:2 5350k -bufsize:v:2 7500k -b:a:2 192k -var_stream_map "v:0,a:0,name:480p v:1,a:1,name:720p v:2,a:2,name:1080p" -master_pl_name master.m3u8 -f hls -hls_time 10 -hls_playlist_type vod -hls_list_size 0 -hls_segment_filename "${rootPath}v%v/segment%d.ts" ${rootPath}v%v/index.m3u8`,
        (err, _, __) => {
          if (err) {
            reject(new Error(err.message));

            return;
          }

          rmSync(rootPath + file.filename);

          resolve();
        },
      );
    });
  }
}
```

# Output
The output will contain the resolution you put in the settings, it will be like this.

```
- public
  - 📁 v480p
    - 📄 segment0.ts
    - 📄 ,,,.ts
    - 📄 segmentx.ts
    - 📄 index.m3u8
  - 📁 v720p
    - 📄 segment0.ts
    - 📄 ,,,.ts
    - 📄 segmentx.ts
    - 📄 index.m3u8
  - 📁 v1080p
    - 📄 segment0.ts
    - 📄 ,,,.ts
    - 📄 segmentx.ts
    - 📄 index.m3u8
  📄 master.m3u8
```

# Upload the videos
As I already said, it depends on the server you will be using, if it can do folder type file storing, you can directly upload it on the server, but if you're using uploadthing or firebase storage you need to change each individual m3u8 to match the url that the storage provider is given.

For started, we need to upload each segments first and store the urls in variable, so we can rewrite each index.m3u8 to a new url, let's see the code below to understand better.

```ts
// app.service.ts
async uploadFiles(fileName: string): Promise<string> {
    const rootPath = __dirname + '/public/';

    const masterPath = rootPath + 'master.m3u8';
    const masterUrl: string[] = [];

    for (const subDir of ['v480p', 'v720p', 'v1080p']) {
      const currentDirPath = rootPath + subDir;
      const indexDirPath = currentDirPath + '/' + 'index.m3u8';

      const filestsPath = this.getTsFiles(currentDirPath);
      const files: File[] = [];

      for (let i = 0; i < filestsPath.length; i++) {
        const buffer = readFileSync(filestsPath[i]);
        files.push(
          new File([buffer], `${fileName}-${subDir}-segment${i}.ts`, {
            type: 'video/ts',
          }),
        );
      }

      const fileRes = await this.utapi.uploadFiles(files);

      const url = this.getFileUrl(fileRes);
      this.replaceSegments(indexDirPath, url);

      const indexBuffer = readFileSync(currentDirPath + '/' + 'index.m3u8');
      const indexRes = await this.utapi.uploadFiles(
        new File([indexBuffer], `${fileName}-${subDir}.m3u8`, {
          type: 'application/x-mpegURL',
        }),
      );

      if (indexRes.data) masterUrl.push(indexRes.data.url);
    }

    this.replacePlaylistName(rootPath + 'master.m3u8', masterUrl);
    const masterBuffer = readFileSync(masterPath);
    const masterRef = await this.utapi.uploadFiles(
      new File([masterBuffer], `${fileName}-master.m3u8`, {
        type: 'application/x-mpegURL',
      }),
    );

    rmSync(rootPath + 'v480p', { recursive: true, force: true });
    rmSync(rootPath + 'v720p', { recursive: true, force: true });
    rmSync(rootPath + 'v1080p', { recursive: true, force: true });

    if (existsSync(rootPath + 'master.m3u8')) rmSync(rootPath + 'master.m3u8');

    if (masterRef.data) return masterRef.data.url;

    throw new Error('master ref empty');
  }
```

Please add your additional resolutions if any for the loop, so it will change and upload the whole files. Also, there are still missing function from the uploadFiles function, these are the function.

> getTsFiles function is using for get all segments in each resolution and return it.

```ts
getTsFiles(dirPath: string): string[] {
    const files: string[] = [];

    const allFiles = readdirSync(dirPath);

    for (const file of allFiles) {
      const filePath = join(dirPath, file);

      if (extname(filePath) === '.ts') {
        files.push(filePath);
      }
    }

    return files;
  }
```

> getFileUrl is a function to reformat the url into string array, which the result come from uploadthing api or utapi.

```ts
getFileUrl(uploadFileResult: UploadFileResult[]): string[] {
    const url: string[] = [];

    for (const res of uploadFileResult) {
      if (res.data) url.push(res.data.url);
    }

    return url;
  }
```

> replaceSegments function will change the m3u8 files for each resolution to match with the url from uploadthing, it's using regex to find and to replace the file location.

```ts
replaceSegments(dirPath: string, urls: string[]) {
    const content = readFileSync(dirPath, 'utf-8');
    const regex = /segment(\d+)\.ts/g;

    const modifiedContent = content.replace(
      regex,
      (match, index) => urls[index],
    );

    writeFileSync(dirPath, modifiedContent, 'utf-8');
  }
```

> replacePlaylistName is a function to replace the url in master.m3u8, if you're using more resolution than I do, please add the missing resolution here.

```ts
replacePlaylistName(dirPath: string, urls: string[]) {
    const content = readFileSync(dirPath, 'utf-8');

    const modifiedContent = content
      .replace('v480p/index.m3u8', urls[0])
      .replace('v720p/index.m3u8', urls[1])
      .replace('v1080p/index.m3u8', urls[2]);

    writeFileSync(dirPath, modifiedContent, 'utf-8');
  }
```
# Closing

It's a quite challenging problem for me, as I never do any backend especially for processing things, but then for now I manage to do it and I have tried it and it works on my project.

I hope this will be useful for you to add video processing feature in your app, as it kinda hard to find source to apply these methods, if you have any question please ask in comment below.

[Github link to the project here.](https://github.com/khouwdevin/tutorial-backend-processing-video)

See ya!