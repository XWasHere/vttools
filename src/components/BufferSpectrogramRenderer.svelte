<script>
    import { onMount } from "svelte";

    export let buf;
    export let bufHeight;
    export let bufLength;
    export let bufInterval;
    let        bufZoomTime = 1;
    let        bufZoomFreq = 1;
    let        bufOffTime  = 0;

    let vw;
    let vh;

    /** @type {HTMLCanvasElement} */
    let cv;

    let ctx;
    let gl_prg;
    let gl_u_abuf;
    let gl_u_bufsiz;
    let gl_u_bufzoom;
    let gl_u_bufoff;
    let gl_a_pos_l;
    let gl_a_pos_b;

    onMount(() => {
        ctx  = cv.getContext("webgl");

        let vs = ctx.createShader(ctx.VERTEX_SHADER);
        ctx.shaderSource(vs, `
attribute vec2 a_pos;

varying vec2 v_pos;

void main() {
    gl_Position = vec4(a_pos, 0, 1);
    v_pos = a_pos;
}`)
        ctx.compileShader(vs);
        if (!ctx.getShaderParameter(vs, ctx.COMPILE_STATUS)) {
            throw ctx.getShaderInfoLog(vs);
        }

        let fs = ctx.createShader(ctx.FRAGMENT_SHADER);
        ctx.shaderSource(fs, `
precision mediump float;

uniform sampler2D u_abuf;

varying vec2 v_pos;

uniform ivec2 u_bufsiz;

uniform vec2 u_bufzoom;
uniform vec2 u_bufoff;

void main() {
    //vec2 onePixel = vec2(2, 2) / u_bufsiz;
    vec2 bufsizf = vec2(u_bufsiz);

    // vec2 rpos = (v_pos + vec2(1, 1)) / vec2(2);

    //vec2 bufImgPos = floor(rpos * bufsizf) / bufsizf; // (v_pos / vec2(1, 1)) * vec2(u_bufsiz);
    //vec4 fvalv     = texture2D(u_abuf, bufImgPos);
    //gl_FragColor   = vec4(fvalv.aaa, 1);
    //gl_FragColor = vec4(v_pos, 0, 1);
    vec2 bufPos  = ((v_pos + 1.0) / 2.0) / u_bufzoom + u_bufoff / vec2(u_bufsiz);

    gl_FragColor = texture2D(u_abuf, bufPos.yx);
}`);
        ctx.compileShader(fs);
        if (!ctx.getShaderParameter(fs, ctx.COMPILE_STATUS)) {
            throw ctx.getShaderInfoLog(fs);
        }

        gl_prg = ctx.createProgram();
        ctx.attachShader(gl_prg, vs);
        ctx.attachShader(gl_prg, fs);
        ctx.linkProgram(gl_prg)
        if (!ctx.getProgramParameter(gl_prg, ctx.LINK_STATUS)) {
            throw ctx.getProgramInfoLog(gl_prg);
        }

        gl_a_pos_l = ctx.getAttribLocation(gl_prg, "a_pos");
        gl_a_pos_b = ctx.createBuffer();
        
        ctx.bindBuffer(ctx.ARRAY_BUFFER, gl_a_pos_b);
        ctx.bufferData(ctx.ARRAY_BUFFER, new Float32Array([
            -1, -1,
            -1,  1,
             1, -1,
             1, -1,
            -1,  1,
             1,  1
        ]), ctx.STATIC_DRAW);
        
        ctx.pixelStorei(ctx.UNPACK_ALIGNMENT, 1);

        gl_u_abuf = ctx.createTexture();
        ctx.bindTexture(ctx.TEXTURE_2D, gl_u_abuf);
        ctx.texImage2D(ctx.TEXTURE_2D, 0, ctx.LUMINANCE, 1, 1, 0, ctx.LUMINANCE, ctx.UNSIGNED_BYTE, new Uint8Array([
            0
        ]));

        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_WRAP_S, ctx.CLAMP_TO_EDGE);
        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_WRAP_T, ctx.CLAMP_TO_EDGE);
        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_MIN_FILTER, ctx.NEAREST);
        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_MAG_FILTER, ctx.NEAREST);

        gl_u_bufsiz  = ctx.getUniformLocation(gl_prg, "u_bufsiz");
        gl_u_bufzoom = ctx.getUniformLocation(gl_prg, "u_bufzoom");
        gl_u_bufoff  = ctx.getUniformLocation(gl_prg, "u_bufoff");

        function redraw() {
            ctx.clearColor(0, 0, 0, 0);
            ctx.clear(ctx.COLOR_BUFFER_BIT);
            ctx.bindTexture(ctx.TEXTURE_2D, gl_u_abuf);
            ctx.activeTexture(ctx.TEXTURE0);
            ctx.texImage2D(ctx.TEXTURE_2D, 0, ctx.LUMINANCE, bufHeight, bufLength, 0, ctx.LUMINANCE, ctx.UNSIGNED_BYTE, buf);
            ctx.useProgram(gl_prg);
            ctx.uniform2i(gl_u_bufsiz, bufLength, bufHeight);
            ctx.uniform2f(gl_u_bufzoom, bufZoomTime, bufZoomFreq);
            ctx.uniform2f(gl_u_bufoff, bufOffTime, 0);
            ctx.enableVertexAttribArray(gl_a_pos_l);
            ctx.bindBuffer(ctx.ARRAY_BUFFER, gl_a_pos_b);
            ctx.vertexAttribPointer(gl_a_pos_l, 2, ctx.FLOAT, false, 0, 0);
            ctx.drawArrays(ctx.TRIANGLES, 0, 6);

            requestAnimationFrame(redraw);
        }

        requestAnimationFrame(redraw);
    });

    function setScrollClamped(time) {
        bufOffTime = Math.min(Math.max(time, 0), bufLength);
    }

    let cont;

    $: if (ctx) {
        ctx.viewport(0, 20, vw, vh - 20);
    }

    let c2;
    /** @type {CanvasRenderingContext2D}*/
    let ctx2;
    function redrawBars() {
        if (c2) {
            requestAnimationFrame(() => {
                if (!ctx2) ctx2 = c2.getContext("2d");

                let frames = bufLength;
                let ssize  = 1000 / bufInterval;
                let scount = frames / ssize;

                let fwidth = vw / bufLength;

                ctx2.clearRect(0, 0, ctx2.canvas.width, ctx2.canvas.height);
                ctx2.textAlign = "left"
                for (let i = 0; i < scount; i++) {
                    let sw = (fwidth * ssize * i * bufZoomTime) - (bufOffTime * fwidth * bufZoomTime);

                    ctx2.beginPath();
                    ctx2.moveTo(sw, vh - 20);
                    ctx2.lineTo(sw, vh);
                    ctx2.stroke();

                    ctx2.fillText(String(i), sw, vh - 2)
                }
            })
        }
    }
    $: (true || vh || vw || bufZoomTime || bufZoomFreq || bufOffTime) && redrawBars();
</script>
<div style:flex-grow="1" bind:this={cont} bind:clientHeight={vh} bind:clientWidth={vw}
     on:wheel={(e)=>{
        e.preventDefault();
        if (e.ctrlKey) {
            bufZoomTime = Math.max(bufZoomTime - e.deltaX / 100, 1);
            bufZoomFreq = Math.max(bufZoomFreq - e.deltaY / 100, 1);
        } else {
            setScrollClamped(bufOffTime + e.deltaX);
        }
     }}>
    <canvas bind:this={cv} height={vh}px width={vw}px></canvas>
    <canvas bind:this={c2} height={vh}px width={vw}px></canvas>
</div>
<style>
    canvas {
        position: fixed;
    }
</style>