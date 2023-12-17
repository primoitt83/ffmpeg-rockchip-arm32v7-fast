# ffmpeg-rockchip-arm32v7-fast

Same as ffmpeg-rockchip-arm32v7 but using pre-compiled libs.

ffmpeg static compiled for arm32v7 with rockchip-mpp, alsa and pulse.

It should work on another arm Rockchip machines too.

Inspired by:

https://github.com/hbiyik/FFmpeg

https://github.com/jjm2473/ffmpeg-rk

https://github.com/wader/static-ffmpeg


## How to build?

- get an armv7 device with Rockchip CPU (tested on rk322x);
- install Armbian;
- install docker;
- install git and then...

````
git clone https://github.com/primoitt83/ffmpeg-rockchip-arm32v7.git

cd ffmpeg-rockchip-arm32v7

docker build --pull --no-cache --rm=true -f Dockerfile -t ffmpeg-rockchip:arm32v7 .
````

## Can I build on x86_64 machine?

Yes, no problem:

````
git clone https://github.com/primoitt83/ffmpeg-rockchip-arm32v7.git

cd ffmpeg-rockchip-arm32v7

docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

docker build --pull --no-cache --rm=true -f Dockerfile_x86 -t ffmpeg:rockchip .
````

## How to test?

Build the image and and shoot ffmpeg from it like this:

Remember to change image name as need it:

````
armv32v7: ffmpeg-rockchip:arm32v7

x86: ffmpeg:rockchip
````

````
docker run -i --rm \
    -w "$PWD" \
    --cap-add=SYS_ADMIN \
    -v $PWD:/tmp \
    --name test \
    ffmpeg-rockchip:arm32v7 \
    ffmpeg -buildconf
````

720p video scale to 360p using CPU (too slow)
````
curl -k http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4 -o BBB.mp4

docker run -i --rm \
    -w "$PWD" \
    --cap-add=SYS_ADMIN \
    -v $PWD:/tmp \
    --name test \
    ffmpeg-rockchip:arm32v7 \
    ffmpeg -y -c:v h264 -i /tmp/BBB.mp4 \
    -an \
    -c:v libx264 -vf scale=w=640:h=360 \
    -f mp4 /tmp/test.mp4
````    

Create hls stream from BBB video (works good on x86 too)

````
docker run -i --rm \
    -w "$PWD" \
    --cap-add=SYS_ADMIN \
    -v $PWD:/tmp \
    --name test \
    ffmpeg-rockchip:arm32v7 \
    ffmpeg -re -stream_loop -1 -i /tmp/BBB.mp4 \
    -c: copy -tune zerolatency \
    -hls_list_size 0 -hls_segment_filename "/tmp/bbb%9d.ts" -hls_playlist_type event \
    -f hls /tmp/bbb.m3u8
````  

720p video scale to 360p using rockchip hardware:

````
docker run -i --rm \
    -w "$PWD" \
    --cap-add=SYS_ADMIN \
    --device=/dev/video0:/dev/video0 \
    --device=/dev/snd:/dev/snd \
        `for dev in iep rga dri dma_heap mpp_service mpp-service vpu_service vpu-service \
            hevc_service hevc-service rkvdec rkvenc avsd vepu h265e ; do \
        [ -e "/dev/$dev" ] && echo " --device /dev/$dev"; \
    done` \
    -v "$PWD":/tmp \
    ffmpeg-rockchip:arm32v7 \
    ffmpeg -y -hwaccel drm -hwaccel_device /dev/dri/renderD128 \
    -c:v h264 -i /tmp/BBB.mp4 \
    -c:a libfdk_aac -strict -2 -ab 64k -ar 44100 -ac 2 \
    -c:v h264_rkmpp_encoder -vf scale=w=640:h=360 -quality_min 4 -quality_max 48 -b:v 1200k -maxrate 1200k -bufsize 2400k -pix_fmt yuv420p \
    /tmp/BBB360p.mkv
````