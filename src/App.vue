<template>
    <div class="app-container">
        <div class="app-wrapper">
            <div class="root content">
                <div class="title">Face Detection</div>
                <div id="camera-container">
                    <camera
                        ref="camera"
                        @loading="logEvent('loading')"
                        @started="logEvent('started')"
                        @error="(error) => logEvent('error: ' + error)"
                        @loadeddata="addListener"
                        autoplay
                    >
                    </camera>
                </div>
                <div>
                    <h1 class="description">
                        A simple application for detecting faces and key points
                        <br />
                        using tensorflow.js and Vue
                    </h1>
                </div>
                <div class="footer-body">
                    <a
                        href="https://github.com/AsakoKabe/face-detection-tfjs-vue"
                        target="_blank"
                    >
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            width="30"
                            height="30"
                            fill="currentColor"
                            class="bi bi-github"
                            viewBox="0 0 16 16"
                        >
                            <path
                                d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.012 8.012 0 0 0 16 8c0-4.42-3.58-8-8-8z"
                            />
                        </svg>
                    </a>
                </div>
            </div>
            <div class="devices">
                <p class="description">Camera device:</p>

                <select @change="changeCamera">
                    <option
                        v-for="camera in cameras"
                        :key="camera"
                        :value="camera.deviceId"
                    >
                        {{ camera.deviceId.slice(0, 20) }}
                    </option>
                </select>
            </div>
        </div>
    </div>
    <!--        <div>-->
    <!--            <button @click="start">Start</button>-->
    <!--            <button @click="stop">Stop</button>-->
    <!--            <button @click="pause">Pause</button>-->
    <!--            <button @click="resume">Resume</button>-->
    <!--            <button @click="snapshot">Snapshot</button>-->
    <!--        </div>-->
</template>

<script lang="ts">
import { defineComponent, onMounted, Ref, ref } from "vue";
import Camera from "@/components/Camera.vue";
import "@tensorflow/tfjs";
import * as blazeface from "@tensorflow-models/blazeface";
import { NormalizedFace } from "@tensorflow-models/blazeface/dist/face";
import * as tf from "@tensorflow/tfjs-core";
import { isMobile } from "@tensorflow/tfjs-core/dist/device_util";

export default defineComponent({
    name: "App",
    components: {
        Camera,
    },
    setup() {
        const camera = ref<InstanceType<typeof Camera>>();

        const cameras: Ref<MediaDeviceInfo[]> = ref([]);

        let ctx: CanvasRenderingContext2D;

        let width = 900;
        let height = 600;
        if (isMobile()) {
            width = 300;
            height = 200;
        }

        let model: Promise<blazeface.BlazeFaceModel>;

        onMounted(async () => {
            if (!camera.value) return;
            try {
                cameras.value = await camera.value.devices(["videoinput"]);
                const res = camera.value?.canvas?.getContext("2d");
                if (!res || !(res instanceof CanvasRenderingContext2D)) {
                    throw new Error("Failed to get 2D context");
                }
                ctx = res;
            } catch (e) {
                console.error(e);
            }
        });

        const start = () => camera.value?.start();
        const stop = () => camera.value?.stop();
        const pause = () => camera.value?.pause();
        const resume = () => camera.value?.resume();
        const snapshot = async () => {
            const blob = await camera.value?.snapshot({
                width: 1280,
                height: 720,
            });
            currentSnapshot.value = URL.createObjectURL(blob!);
        };

        const logEvent = (name: string) => console.log(name);

        const currentSnapshot = ref();

        const changeCamera = (event: Event) => {
            const target = event.target as HTMLSelectElement;
            camera.value?.changeCamera(target.value);
        };

        const detectFaces = async () => {
            let video: HTMLVideoElement | undefined = camera.value?.video;
            if (!video || !(video instanceof HTMLVideoElement)) {
                throw new Error("Failed to get 2D context");
            }
            ctx.canvas.width = width;
            ctx.canvas.height = height;
            ctx.drawImage(video, 0, 0, width, height);

            (await model)
                .estimateFaces(video, false)
                .then((prediction: Array<NormalizedFace>) => {
                    prediction.forEach((pred: any) => {
                        // draw the rectangle enclosing the face
                        ctx.beginPath();
                        ctx.lineWidth = Number("4");
                        ctx.strokeStyle = "blue";
                        // the last two arguments are width and height
                        // since blazeface returned only the coordinates,
                        // we can find the width and height by subtracting them.
                        ctx.rect(
                            pred.topLeft[0],
                            pred.topLeft[1],
                            pred.bottomRight[0] - pred.topLeft[0],
                            pred.bottomRight[1] - pred.topLeft[1]
                        );
                        ctx.stroke();

                        // drawing small rectangles for the face landmarks
                        ctx.fillStyle = "red";
                        pred.landmarks.forEach((landmark: any) => {
                            ctx.fillRect(landmark[0], landmark[1], 5, 5);
                        });
                    });
                });
        };

        const addListener = async () => {
            model = blazeface.load();
            setInterval(detectFaces, 100);
        };

        return {
            camera,
            start,
            stop,
            pause,
            resume,
            logEvent,
            snapshot,
            currentSnapshot,
            cameras,
            changeCamera,
            addListener,
        };
    },
});
</script>

<style>
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
    width: 100%;
    height: 100%;
}
.app-container {
    top: 0;
    left: 0;
    height: 100vh;
    width: 100vw;
    background-color: white;
    position: fixed;
    transition: 0.2s;
    z-index: 100;
    flex-grow: 1;
}

.app-wrapper {
    width: 100%;
    height: 100%;
    position: relative;
    display: flex;
    flex-direction: row;
    overflow: auto;
}

.root {
    /*flex-grow: 1;*/
    flex-basis: 80%;
}

.title {
    /*color: #26ade4;*/
    font-size: 32px;
    font-weight: 700;
    overflow: hidden;
    white-space: nowrap;
    letter-spacing: 0.15em;
    animation: typing 0.5s steps(13, end);
    padding: 20px;
}
.description {
    text-align: center;
    font-size: 25px;
    margin-bottom: 20px;
}

.footer-body {
    padding: 20px;
    font-weight: bold;
    max-width: 1000px;
    /*margin: 0 auto;*/
    display: flex;
    flex-direction: row;
    justify-content: space-between;
}
.content {
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
}
.devices {
    justify-content: center;
    align-items: center;

    display: flex;
    flex-direction: column;
    /*width: 10%;*/
    /*padding-right: 20%;*/
}
</style>
