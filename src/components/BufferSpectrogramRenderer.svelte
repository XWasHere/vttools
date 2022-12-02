<script>
    import { onMount } from "svelte";

    export let buf;
    export let bufHeight;
    export let bufLength;

    let vw;
    let vh;

    /** @type {HTMLCanvasElement} */
    let cv;

    let ctx;

    onMount(() => {
        ctx = cv.getContext("webgl");

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

void main() {
    //vec2 onePixel = vec2(2, 2) / u_bufsiz;
    vec2 bufsizf = vec2(u_bufsiz);

    // vec2 rpos = (v_pos + vec2(1, 1)) / vec2(2);

    //vec2 bufImgPos = floor(rpos * bufsizf) / bufsizf; // (v_pos / vec2(1, 1)) * vec2(u_bufsiz);
    //vec4 fvalv     = texture2D(u_abuf, bufImgPos);
    //gl_FragColor   = vec4(fvalv.aaa, 1);
    //gl_FragColor = vec4(v_pos, 0, 1);
    vec2 bufPos  = (v_pos + 1.0) / 2.0;

    gl_FragColor = texture2D(u_abuf, bufPos.yx);
}`);
        ctx.compileShader(fs);
        if (!ctx.getShaderParameter(fs, ctx.COMPILE_STATUS)) {
            throw ctx.getShaderInfoLog(fs);
        }

        let pg = ctx.createProgram();
        ctx.attachShader(pg, vs);
        ctx.attachShader(pg, fs);
        ctx.linkProgram(pg);
        if (!ctx.getProgramParameter(pg, ctx.LINK_STATUS)) {
            throw ctx.getProgramInfoLog(pg);
        }

        let a_pos_l = ctx.getAttribLocation(pg, "a_pos");
        let a_pos_b = ctx.createBuffer();
        
        ctx.bindBuffer(ctx.ARRAY_BUFFER, a_pos_b);
        ctx.bufferData(ctx.ARRAY_BUFFER, new Float32Array([
            -1, -1,
            -1,  1,
             1, -1,
             1, -1,
            -1,  1,
             1,  1
        ]), ctx.STATIC_DRAW);
        
        ctx.pixelStorei(ctx.UNPACK_ALIGNMENT, 1);

        let u_abuf_t = ctx.createTexture();
        ctx.bindTexture(ctx.TEXTURE_2D, u_abuf_t);
        ctx.texImage2D(ctx.TEXTURE_2D, 0, ctx.LUMINANCE, 1, 1, 0, ctx.LUMINANCE, ctx.UNSIGNED_BYTE, new Uint8Array([
            0
        ]));

        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_WRAP_S, ctx.CLAMP_TO_EDGE);
        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_WRAP_T, ctx.CLAMP_TO_EDGE);
        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_MIN_FILTER, ctx.NEAREST);
        ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_MAG_FILTER, ctx.NEAREST);

        let u_bufsiz_l = ctx.getUniformLocation(pg, "u_bufsiz");
        setInterval(() => {
            ctx.clearColor(0, 0, 0, 0);
            ctx.clear(ctx.COLOR_BUFFER_BIT);
            ctx.bindTexture(ctx.TEXTURE_2D, u_abuf_t);
            ctx.activeTexture(ctx.TEXTURE0);
            ctx.texImage2D(ctx.TEXTURE_2D, 0, ctx.LUMINANCE, bufHeight, bufLength, 0, ctx.LUMINANCE, ctx.UNSIGNED_BYTE, buf);
            ctx.useProgram(pg);
            ctx.uniform2i(u_bufsiz_l, bufLength, bufHeight);
            ctx.enableVertexAttribArray(a_pos_l);
            ctx.bindBuffer(ctx.ARRAY_BUFFER, a_pos_b);
            ctx.vertexAttribPointer(a_pos_l, 2, ctx.FLOAT, false, 0, 0);
            ctx.drawArrays(ctx.TRIANGLES, 0, 6);
        }, 100);
    });

    let cont;

    $: if (ctx) ctx.viewport(0, 0, vw, vh);
</script>
<div style:flex-grow="1" bind:this={cont} bind:clientHeight={vh} bind:clientWidth={vw}>
    <canvas bind:this={cv} height={vh}px width={vw}px></canvas>
</div>
<style>
    canvas {
        position: fixed;
    }
</style>