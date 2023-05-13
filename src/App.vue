<script lang='ts'>
import { defineComponent } from 'vue';

export default defineComponent({
    data() {
        return {
            events: [] as string[]
        };
    },
    methods: {
        addEvent(e: string) {
            this.events.push(e);
        },
        capture() {
            const stats = {
                type: null,
                bytes: 0
            };

            const handle = (devices: MediaStream[]) => {
                this.addEvent(`${ new Date() }: Acquiring media devices`);

                const acquired = devices.map(device => {
                    const track = device.getVideoTracks()[0];
                    const capabilities = track.getCapabilities();

                    return {
                        track,
                        info: {
                            device: {
                                id: capabilities.deviceId,
                                type: capabilities.displaySurface === 'monitor' ? 'screen' : 'camera',
                                kind: track.kind,
                                label: track.label,
                                state: track.readyState,
                                fps: capabilities.frameRate!.max
                            },
                            dimensions: {
                                width: capabilities.width!.max,
                                height: capabilities.height!.max
                            }
                        }
                    };
                });

                this.addEvent(`${ new Date() }: Tracks acquired successfuly: ${ acquired.length }/${ devices.length }`);

                const screen = getAcquiredTrack(acquired, 'screen');
                const camera = getAcquiredTrack(acquired, 'camera');

                const stream = new MediaStream([ screen.track, camera.track ]);
                const chunks = [];

                const recorder = new MediaRecorder(stream);

                recorder.ondataavailable = e => {
                    chunks.push(e.data);
                    stats.bytes += e.data.size;
                    (this.$refs.bytes as HTMLElement).innerHTML = stats.bytes.toString();

                    console.log(e.data, `${ chunks.length } saved frames`);
                };

                recorder.onstop = e => {
                    this.addEvent(`${ new Date() }: Stopping..`);

                    // const blob = new Blob(chunks);
                    // const url = URL.createObjectURL(blob);
                    // const c = url.cloneNode();
                    // c.loop = true;
                    // v.replaceWith(c);
                    // c.src = url;
                    // c.play();

                    this.addEvent(`${ new Date() }: Final Blob object created containing ${ chunks.length } frames`);

                    stop(recorder.stream);
                };

                recorder.onstart = () => {
                    this.addEvent(`${ new Date() }: Start event triggered by MediaRecorder!`);
                };

                recorder.start(30);

                /*v.srcObject = stream; v.play(); */

                //
                // Artifically stop recording after 5-seconds.
                // This would ideally be triggered by the user or other process(es).
                //
                setTimeout(() => recorder.stop(), 5000);
            };

            const getAcquiredTrack = (acquired: any, type: string) => {
                return acquired.find((track: any) => track.info.device.type === type);
            };

            const stop = (stream: MediaStream) => {
                stream.getTracks().forEach(track => track.stop());
            };

            Promise.all([
                navigator.mediaDevices.getUserMedia({
                    video: true,
                    audio: false
                }),
                navigator.mediaDevices.getDisplayMedia({
                    video: true,
                    audio: false
                })
            ]).then(handle);
        }
    },
    mounted() {

    }
});
</script>

<template>
    <button @click='capture'>Start recording..</button>

    <div>Total bytes processed: <span ref='bytes'></span></div>
    <div>
        <h4>Events</h4>
        <ul>
            <li v-for='event in events'
                ref='events'>{{ event }}
            </li>
        </ul>
    </div>
</template>
