<script lang="ts">
  import type { Snippet } from "svelte";

  type Props = {
    children: Snippet;
    bloom?: number;
    scanlineSize?: number;
    scanlineDepth?: number;
    scanlineSpeed?: number;
    flicker?: number;
    warp?: number;
    chromatic?: number;
  };

  let {
    children,
    bloom = 0.98,
    scanlineSize = 5,
    scanlineDepth = 0.46,
    scanlineSpeed = 8,
    flicker = 0.02,
    warp = 0.075,
    chromatic = 1.35,
  }: Props = $props();
  let source: HTMLCanvasElement;
  let content: HTMLDivElement;
  let output: HTMLCanvasElement;
  let supported = $state(true);
  const layoutSubtree = { layoutsubtree: "true" };

  const vertexShader = `#version 300 es
in vec2 aPosition;
out vec2 vUv;
void main() {
  vUv = aPosition * .5 + .5;
  gl_Position = vec4(aPosition, 0., 1.);
}`;

  const fragmentShader = `#version 300 es
precision highp float;
in vec2 vUv;
out vec4 fragColor;
uniform sampler2D uScreen;
uniform vec2 uResolution;
uniform float uTime;
uniform float uBloom;
uniform float uScanlineSize;
uniform float uScanlineDepth;
uniform float uScanlineSpeed;
uniform float uFlicker;
uniform float uWarp;
uniform float uChromatic;
vec2 curve(vec2 uv) {
  vec2 p = uv * 2. - 1.;
  float r2 = dot(p, p);
  p *= 1. + r2 * uWarp;
  return p * .5 + .5;
}
float rand(vec2 p) { return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453); }
void main() {
  vec2 uv = curve(vUv);
  vec2 edge = smoothstep(vec2(0.), vec2(.035), uv) * smoothstep(vec2(0.), vec2(.035), 1. - uv);
  float mask = edge.x * edge.y;
  if (mask <= 0.) { fragColor = vec4(0., 0., 0., 1.); return; }
  vec2 px = 1. / uResolution;
  float aberration = uChromatic * px.x;
  vec3 base;
  base.r = texture(uScreen, uv + vec2(aberration, 0.)).r;
  base.g = texture(uScreen, uv).g;
  base.b = texture(uScreen, uv - vec2(aberration, 0.)).b;
  vec3 bloom = vec3(0.);
  bloom += texture(uScreen, uv + px * vec2(-5., 0.)).rgb;
  bloom += texture(uScreen, uv + px * vec2(5., 0.)).rgb;
  bloom += texture(uScreen, uv + px * vec2(0., -4.)).rgb;
  bloom += texture(uScreen, uv + px * vec2(0., 4.)).rgb;
  bloom += texture(uScreen, uv + px * vec2(-3., -3.)).rgb;
  bloom += texture(uScreen, uv + px * vec2(3., -3.)).rgb;
  bloom += texture(uScreen, uv + px * vec2(-3., 3.)).rgb;
  bloom += texture(uScreen, uv + px * vec2(3., 3.)).rgb;
  bloom *= .125;
  float bloomLuma = max(max(bloom.r, bloom.g), bloom.b);
  vec3 color = base + bloom * smoothstep(.14, .74, bloomLuma) * uBloom;
  float scanPhase = abs(fract((gl_FragCoord.y + uTime * uScanlineSpeed) / uScanlineSize) - .5);
  float scan = mix(1. - uScanlineDepth, 1., smoothstep(.12, .34, scanPhase));
  float grille = .94 + .06 * sin(uv.x * uResolution.x * 2.094);
  float fastFlicker = 1. - uFlicker + uFlicker * sin(uTime * 43. + sin(uTime * 9.) * 1.5);
  float randomFlicker = 1. - step(.985, rand(vec2(floor(uTime * 14.), .37))) * uFlicker * 2.25;
  float flicker = fastFlicker * randomFlicker;
  float vignette = pow(16. * uv.x * uv.y * (1. - uv.x) * (1. - uv.y), .16);
  color *= scan * grille * flicker * vignette * mask;
  color *= vec3(1.0, 1.035, 1.0);
  fragColor = vec4(max(color, 0.), 1.);
}`;

  type PaintableCanvas = HTMLCanvasElement & {
    onpaint?: (() => void) | null;
    requestPaint?: () => void;
  };
  type ElementImageContext = CanvasRenderingContext2D & {
    drawElementImage?: (element: Element, x: number, y: number) => void;
  };

  function compile(gl: WebGL2RenderingContext, type: number, shaderSource: string) {
    const shader = gl.createShader(type);
    if (!shader) throw new Error("Unable to create CRT shader");
    gl.shaderSource(shader, shaderSource);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
      throw new Error(gl.getShaderInfoLog(shader) ?? "CRT shader failed to compile");
    }
    return shader;
  }

  $effect(() => {
    const paintable = source as PaintableCanvas;
    const sourceContext = source.getContext("2d") as ElementImageContext | null;
    const available = Boolean(sourceContext?.drawElementImage && paintable.requestPaint);
    supported = available;
    if (!available || !sourceContext) return;

    const gl = output.getContext("webgl2", { alpha: false, antialias: false, depth: false, premultipliedAlpha: false });
    if (!gl) { supported = false; return; }
    const vert = compile(gl, gl.VERTEX_SHADER, vertexShader);
    const frag = compile(gl, gl.FRAGMENT_SHADER, fragmentShader);
    const program = gl.createProgram();
    if (!program) return;
    gl.attachShader(program, vert);
    gl.attachShader(program, frag);
    gl.linkProgram(program);
    gl.useProgram(program);
    const buffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1]), gl.STATIC_DRAW);
    const position = gl.getAttribLocation(program, "aPosition");
    gl.enableVertexAttribArray(position);
    gl.vertexAttribPointer(position, 2, gl.FLOAT, false, 0, 0);
    const texture = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
    const resolution = gl.getUniformLocation(program, "uResolution");
    const time = gl.getUniformLocation(program, "uTime");
    const bloomLocation = gl.getUniformLocation(program, "uBloom");
    const scanlineSizeLocation = gl.getUniformLocation(program, "uScanlineSize");
    const scanlineDepthLocation = gl.getUniformLocation(program, "uScanlineDepth");
    const scanlineSpeedLocation = gl.getUniformLocation(program, "uScanlineSpeed");
    const flickerLocation = gl.getUniformLocation(program, "uFlicker");
    const warpLocation = gl.getUniformLocation(program, "uWarp");
    const chromaticLocation = gl.getUniformLocation(program, "uChromatic");
    let dirty = true;
    let frame = 0;
    const resize = () => {
      const dpr = Math.min(window.devicePixelRatio || 1, 2);
      output.width = Math.max(1, Math.round(output.clientWidth * dpr));
      output.height = Math.max(1, Math.round(output.clientHeight * dpr));
      paintable.width = output.width;
      paintable.height = output.height;
      paintable.requestPaint?.();
      dirty = true;
    };
    paintable.onpaint = () => {
      sourceContext.reset();
      sourceContext.scale(output.width / Math.max(output.clientWidth, 1), output.height / Math.max(output.clientHeight, 1));
      sourceContext.drawElementImage?.(content, 0, 0);
      dirty = true;
    };
    const startedAt = performance.now();
    const render = (now: number) => {
      if (dirty) {
        gl.bindTexture(gl.TEXTURE_2D, texture);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, paintable);
        dirty = false;
      }
      gl.viewport(0, 0, output.width, output.height);
      gl.uniform2f(resolution, output.width, output.height);
      gl.uniform1f(time, (now - startedAt) / 1000);
      gl.uniform1f(bloomLocation, bloom);
      gl.uniform1f(scanlineSizeLocation, scanlineSize);
      gl.uniform1f(scanlineDepthLocation, scanlineDepth);
      gl.uniform1f(scanlineSpeedLocation, scanlineSpeed);
      gl.uniform1f(flickerLocation, flicker);
      gl.uniform1f(warpLocation, warp);
      gl.uniform1f(chromaticLocation, chromatic);
      gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
      frame = requestAnimationFrame(render);
    };
    const mappedPoint = (event: PointerEvent | MouseEvent) => {
      const rect = output.getBoundingClientRect();
      const x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      const y = ((event.clientY - rect.top) / rect.height) * 2 - 1;
      const scale = 1 + (x * x + y * y) * warp;
      return {
        x: rect.left + ((x * scale + 1) * 0.5) * rect.width,
        y: rect.top + ((y * scale + 1) * 0.5) * rect.height,
      };
    };
    const interactiveSelector = "button, a, input, select, textarea, [role='button']";
    const interactiveAt = (event: PointerEvent | MouseEvent) => {
      const point = mappedPoint(event);
      const elements = Array.from(content.querySelectorAll<HTMLElement>(interactiveSelector));
      return elements.reverse().find((element) => {
        const rect = element.getBoundingClientRect();
        return point.x >= rect.left && point.x <= rect.right && point.y >= rect.top && point.y <= rect.bottom;
      }) ?? null;
    };
    let draggedRange: HTMLInputElement | null = null;
    let hovered: HTMLElement | null = null;
    const updateHover = (event: PointerEvent) => {
      const next = interactiveAt(event);
      if (next === hovered) return;
      hovered?.classList.remove("crt-hover");
      hovered = next;
      hovered?.classList.add("crt-hover");
      source.style.cursor = hovered ? "pointer" : "default";
    };
    const updateRange = (input: HTMLInputElement, event: PointerEvent) => {
      const point = mappedPoint(event);
      const rect = input.getBoundingClientRect();
      const ratio = Math.min(1, Math.max(0, (point.x - rect.left) / rect.width));
      const min = Number(input.min || 0);
      const max = Number(input.max || 100);
      const step = Number(input.step || 1);
      input.valueAsNumber = Math.round((min + ratio * (max - min)) / step) * step;
      input.dispatchEvent(new Event("input", { bubbles: true }));
    };
    const onPointerDown = (event: PointerEvent) => {
      const target = interactiveAt(event);
      if (target instanceof HTMLInputElement && target.type === "range") {
        event.preventDefault();
        event.stopPropagation();
        draggedRange = target;
        updateRange(target, event);
        source.setPointerCapture(event.pointerId);
      }
    };
    const onPointerMove = (event: PointerEvent) => {
      updateHover(event);
      if (draggedRange) updateRange(draggedRange, event);
    };
    const onPointerUp = () => { draggedRange = null; };
    const onPointerLeave = () => {
      hovered?.classList.remove("crt-hover");
      hovered = null;
      source.style.cursor = "default";
    };
    const onClick = (event: MouseEvent) => {
      if (!event.isTrusted) return;
      const target = interactiveAt(event);
      if (!target) return;
      event.preventDefault();
      event.stopPropagation();
      target.click();
    };
    source.addEventListener("pointerdown", onPointerDown, true);
    source.addEventListener("pointermove", onPointerMove, true);
    source.addEventListener("pointerup", onPointerUp, true);
    source.addEventListener("pointercancel", onPointerUp, true);
    source.addEventListener("pointerleave", onPointerLeave, true);
    source.addEventListener("click", onClick, true);

    const observer = new ResizeObserver(resize);
    observer.observe(output);
    resize();
    frame = requestAnimationFrame(render);
    return () => {
      cancelAnimationFrame(frame);
      observer.disconnect();
      source.removeEventListener("pointerdown", onPointerDown, true);
      source.removeEventListener("pointermove", onPointerMove, true);
      source.removeEventListener("pointerup", onPointerUp, true);
      source.removeEventListener("pointercancel", onPointerUp, true);
      source.removeEventListener("pointerleave", onPointerLeave, true);
      source.removeEventListener("click", onClick, true);
      hovered?.classList.remove("crt-hover");
      paintable.onpaint = null;
      gl.deleteTexture(texture);
      gl.deleteBuffer(buffer);
      gl.deleteProgram(program);
      gl.deleteShader(vert);
      gl.deleteShader(frag);
    };
  });
</script>

<div class:crt-native={supported} class:crt-fallback={!supported} class="crt-shell">
  <canvas bind:this={source} {...layoutSubtree} class="crt-source">
    <div bind:this={content} class="crt-content">{@render children()}</div>
  </canvas>
  {#if !supported}
    <div class="crt-content crt-content-fallback">{@render children()}</div>
  {/if}
  <canvas bind:this={output} class="crt-output" aria-hidden="true"></canvas>
  {#if !supported}<div class="fallback-effects" aria-hidden="true"></div>{/if}
</div>
