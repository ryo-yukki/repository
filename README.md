<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <title>虹色のつやつき円が降る WebGL</title>
  <style>
    body { margin: 0; background: black; overflow: hidden; }
    canvas { display: block; }
  </style>
</head>
<body>
  <canvas id="canvas" width="800" height="1200"></canvas>
  <script type="module">
    const canvas = document.getElementById("canvas");
    const gl = canvas.getContext("webgl");
    if (!gl) alert("WebGL が使えません");

    // 頂点シェーダ
    const vertexShaderSource = `
      attribute vec2 a_position;
      attribute vec3 a_color;
      uniform vec2 u_resolution;
      uniform float u_pointSize;
      varying vec3 v_color;

      void main() {
        vec2 zeroToOne = a_position / u_resolution;
        vec2 zeroToTwo = zeroToOne * 2.0;
        vec2 clipSpace = zeroToTwo - 1.0;
        gl_Position = vec4(clipSpace * vec2(1, -1), 0, 1);
        gl_PointSize = u_pointSize;
        v_color = a_color;
      }
    `;

    // フラグメントシェーダ（つや付き＆半透明）
    const fragmentShaderSource = `
      precision mediump float;
      varying vec3 v_color;

      void main() {
        vec2 coord = gl_PointCoord - vec2(0.5);
        float dist = length(coord);
        if (dist > 0.5) discard;

        vec3 baseColor = v_color;
	// 色の半透明度
        float alpha = 0.8;

        // つや（左上からのハイライト）
        vec2 lightDir = normalize(vec2(-0.3, -0.3));
        float highlight = dot(normalize(coord), lightDir);
        float glow = smoothstep(0.2, 0.5, highlight);

        vec3 color = baseColor + vec3(glow * 0.5);
        gl_FragColor = vec4(color, alpha);
      }
    `;

    function createShader(gl, type, source) {
      const shader = gl.createShader(type);
      gl.shaderSource(shader, source);
      gl.compileShader(shader);
      return shader;
    }

    function createProgram(gl, vs, fs) {
      const program = gl.createProgram();
      gl.attachShader(program, vs);
      gl.attachShader(program, fs);
      gl.linkProgram(program);
      return program;
    }

    const vs = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource);
    const fs = createShader(gl, gl.FRAGMENT_SHADER, fragmentShaderSource);
    const program = createProgram(gl, vs, fs);

    const aPositionLoc = gl.getAttribLocation(program, "a_position");
    const aColorLoc = gl.getAttribLocation(program, "a_color");
    const uResolutionLoc = gl.getUniformLocation(program, "u_resolution");
    const uPointSizeLoc = gl.getUniformLocation(program, "u_pointSize");

    const positionBuffer = gl.createBuffer();
    const colorBuffer = gl.createBuffer();

    let drops = [];
    let frameCount = 0;

    function createDrop() {
      return {
        x: Math.random() * canvas.width,
        y: 0,
	// 円の大きさランダム 10 - 20
        r: 10 + Math.random() * 40,
        color: hsvToRgb(Math.random(), 1, 1),
      };
    }

    function hsvToRgb(h, s, v) {
      let f = (n, k = (n + h * 6) % 6) =>
        v - v * s * Math.max(Math.min(k, 4 - k, 1), 0);
      return [f(5), f(3), f(1)];
    }

    function update() {
      frameCount++;
      // 15フレームに1個、最大50個まで
      if (frameCount % 5 === 0 && drops.length < 300) {
        drops.push(createDrop());
      }
     // 降ってくるスピード
      drops.forEach(d => d.y += 3.0);
      drops = drops.filter(d => d.y < canvas.height);
    }

    function render() {
      gl.viewport(0, 0, canvas.width, canvas.height);
      gl.clearColor(0, 0, 0, 1);
      gl.clear(gl.COLOR_BUFFER_BIT);
      gl.useProgram(program);
      gl.uniform2f(uResolutionLoc, canvas.width, canvas.height);

      const positions = [];
      const colors = [];

      drops.forEach(d => {
        positions.push(d.x, d.y);
        colors.push(...d.color);
      });

      gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
      gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.DYNAMIC_DRAW);
      gl.enableVertexAttribArray(aPositionLoc);
      gl.vertexAttribPointer(aPositionLoc, 2, gl.FLOAT, false, 0, 0);

      gl.bindBuffer(gl.ARRAY_BUFFER, colorBuffer);
      gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(colors), gl.DYNAMIC_DRAW);
      gl.enableVertexAttribArray(aColorLoc);
      gl.vertexAttribPointer(aColorLoc, 3, gl.FLOAT, false, 0, 0);

      drops.forEach((d, i) => {
        gl.uniform1f(uPointSizeLoc, d.r * 2);
        gl.drawArrays(gl.POINTS, i, 1);
      });
    }

    gl.enable(gl.BLEND);
    gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);

    function loop() {
      update();
      render();
      requestAnimationFrame(loop);
    }

    loop();
  </script>
</body>
</html>
