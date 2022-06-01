<template>
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

<!--        <div>-->
<!--            <button @click="start">Start</button>-->
<!--            <button @click="stop">Stop</button>-->
<!--            <button @click="pause">Pause</button>-->
<!--            <button @click="resume">Resume</button>-->
<!--            <button @click="snapshot">Snapshot</button>-->
<!--        </div>-->

    <!--    <select @change="changeCamera">-->
    <!--        <option-->
    <!--            v-for="camera in cameras"-->
    <!--            :key="camera"-->
    <!--            :value="camera.deviceId"-->
    <!--        >-->
    <!--            {{ camera.deviceId }}-->
    <!--        </option>-->
    <!--    </select>-->
</template>

<script lang="ts">
import { defineComponent, onMounted, Ref, ref } from "vue";
import Camera from "@/components/Camera.vue";
import "@tensorflow/tfjs";
import * as blazeface from "@tensorflow-models/blazeface";

export default defineComponent({
    name: "App",
    components: {
        Camera,
    },
    setup() {
        const camera = ref<InstanceType<typeof Camera>>();

        const cameras: Ref<MediaDeviceInfo[]> = ref([]);

        let ctx: CanvasRenderingContext2D;

        const width = 600;
        const height = 400;

        let model: any;

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

            let prediction: any;
            (await model).estimateFaces(video, false).then((res: any) => {
                prediction = res;
                console.log(prediction, video?.srcObject);

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

<style scoped>
#camera-container {
    width: 300px;
    height: 300px;
}
</style>
