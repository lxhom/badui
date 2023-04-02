<!--suppress TypeScriptUnresolvedReference -->
<script lang="ts">
    import {onMount} from "svelte";
    import {scalePow} from "d3";

    let volume1 = 0.5
    $: volume100 = Math.round(volume1 * 100)
    let volume_scaled = 0.5

    let volumeScale = scalePow()
        .exponent(2)
        .domain([0, 1])
        .range([0, 1]);

    $: volume_scaled = volumeScale(volume1)

    let dbg = ""

    let audio: HTMLAudioElement = <any>null

    let started = false
    let start = () => {
        calibrate()
        started = true
        audio.play()
    }

    let calibrate = () => {}

    onMount(async () => {
        if (!(await Promise.all([
            navigator.permissions.query(<PermissionDescriptor>{name: "gyroscope"}),
        ])).every(({state}) => state === "granted")) {
            return alert("Permission to use sensor was denied.");
        }
        if (typeof window.Gyroscope === 'undefined') {
            dbg = ('Gyroscope is not supported')
            return
        }

        try {
            let gyroscope = new Gyroscope({frequency: 120});
        } catch (e) {
            dbg = ('Gyroscope is not allowed :( Error: ' + e)
        }
        let cum = 0


        gyroscope.addEventListener("reading", (e) => {
            dbg = 'xyz'.split('').map((c, i) => `${c}: ${gyroscope[c].toFixed(2)}`).join(' ')
            cum -= (gyroscope.z * 0.005)
            volume1 = Math.max(Math.min(volume1 + cum, 1), 0)
        });

        calibrate = () => {
            cum = 0
            volume1 = 0.5
        }

        gyroscope.start();

        setTimeout(() => {
            if (!gyroscope.x) {
                dbg = 'Uh oh. No gyroscope data :('
            }
        })
    })
</script>

<div class="h-full flex justify-center items-center flex-col">
    Volume: {volume100}%
    <progress id="file" max="100" value={volume100} class="progress progress-primary w-56 mb-2"></progress>
    <audio bind:volume={volume_scaled} bind:this={audio} loop>
        <source src="https://s3-us-west-2.amazonaws.com/true-commitment/01-NeverGonnaGiveYouUp.mp3" type="audio/mpeg" >
    </audio>
    <div>
        {#if !started}
            <button class="btn btn-primary" on:click={start}>Start</button>
        {/if}
        <button class="btn btn-primary" on:click={calibrate}>Calibrate</button>
    </div>
    {dbg}
</div>
