Title: WebGL2 Множественные виды, множественные canvas
Description: Рисование множественных видов
TOC: Множественные виды, множественные canvas

Эта статья предполагает, что вы прочитали статью о
[меньше кода больше веселья](webgl-less-code-more-fun.html),
поскольку она использует библиотеку, упомянутую там, чтобы
не загромождать пример. Если вы не понимаете,
что такое буферы, вершинные массивы и атрибуты, или когда
функция с именем `twgl.setUniforms` что это означает
устанавливать uniform'ы, и т.д... тогда вам, вероятно, стоит пойти дальше назад и
[прочитать основы](webgl-fundamentals.html).

Допустим, вы хотели нарисовать множественные виды
той же сцены, как мы могли бы это сделать? Один способ был бы
[рендерить в текстуры](webgl-render-to-texture.html),
а затем рисовать эти текстуры на canvas. Это
определенно правильный способ сделать это, и есть времена, когда это
может быть правильной вещью для делания. Но это требует, чтобы мы
выделили текстуры, отрендерили вещи в них, затем отрендерили
эти текстуры на canvas. Это означает, что мы эффективно
двойной рендеринг. Это может быть подходящим, например,
в гоночной игре, когда мы хотим отрендерить вид в зеркале
заднего вида, мы бы отрендерили то, что позади машины, в текстуру,
затем использовали бы эту текстуру для рисования зеркала заднего вида.

Другой способ - это установить viewport и включить scissor тест.
Это отлично для ситуаций, где наши виды не перекрываются. Еще
лучше нет двойного рендеринга, как в решении выше.

В [самой первой статье](webgl-fundamentals.html) упоминается,
что мы устанавливаем, как WebGL конвертирует из пространства отсечения в пространство пикселей, вызывая

```js
gl.viewport(left, bottom, width, height);
```

Самая распространенная вещь - это установить их в `0`, `0`, `gl.canvas.width` и `gl.canvas.height`
соответственно, чтобы покрыть весь canvas.

Вместо этого мы можем установить их в часть canvas, и они сделают так,
что мы будем рисовать только в этой части canvas.
WebGL обрезает вершины в пространстве отсечения.
Как мы упоминали раньше, мы устанавливаем `gl_Position` в нашем вершинном шейдере в значения, которые идут от -1 до +1 в x, y, z.
WebGL обрезает треугольники и линии, которые мы передаем, к этому диапазону. После того, как происходит обрезка, затем
применяются настройки `gl.viewport`, так что, например, если мы использовали

```js
gl.viewport(
   10,   // left
   20,   // bottom
   30,   // width
   40,   // height
);
```

Тогда значение пространства отсечения x = -1 соответствует пикселю x = 10, а значение пространства отсечения
+1 соответствует пикселю x = 40 (left 10 плюс width 30)
(На самом деле это небольшое упрощение, [см. ниже](#pixel-coords))

Итак, после обрезки, если мы рисуем треугольник, он появится, чтобы поместиться внутри viewport.

Давайте нарисуем нашу 'F' из [предыдущих статей](webgl-3d-perspective.html).

Вершинный и фрагментный шейдеры те же, что используются в статьях о
[ортографической](webgl-3d-orthographic.html) и [перспективной](webgl-3d-perspective.html)
проекции.

```glsl
#version 300 es
// вершинный шейдер
in vec4 a_position;
in vec4 a_color;

uniform mat4 u_matrix;

out vec4 v_color;

void main() {
  // Умножаем позицию на матрицу.
  gl_Position = u_matrix * a_position;

  // Передаем цвет вершины в фрагментный шейдер.
  v_color = a_color;
}
```

```glsl
#version 300 es
// фрагментный шейдер
precision highp float;

// Передается из вершинного шейдера.
in vec4 v_color;

out vec4 outColor;

void main() {
  outColor = v_color;
}
```

Затем во время инициализации нам нужно создать программу и
буферы и вершинный массив для 'F'

```js
// настройка GLSL программ
// компилирует шейдеры, связывает программу, ищет расположения
const programInfo = twgl.createProgramInfo(gl, [vs, fs]);

// Говорим twgl сопоставить position с a_position,
// normal с a_normal и т.д..
twgl.setAttributePrefix("a_");

// создаем буферы и заполняем данными для 3D 'F'
const bufferInfo = twgl.primitives.create3DFBufferInfo(gl);
const vao = twgl.createVAOFromBufferInfo(gl, programInfo, bufferInfo);
```

И для рисования давайте сделаем функцию, которой мы можем передать матрицу проекции,
матрицу камеры и мировую матрицу

```js
function drawScene(projectionMatrix, cameraMatrix, worldMatrix) {
  // Делаем view матрицу из матрицы камеры.
  const viewMatrix = m4.inverse(cameraMatrix);

  let mat = m4.multiply(projectionMatrix, viewMatrix);
  mat = m4.multiply(mat, worldMatrix);

  gl.useProgram(programInfo.program);

  // ------ Рисуем F --------

  // Настраиваем все нужные атрибуты.
  gl.bindVertexArray(vao);

  // Устанавливаем uniform'ы
  twgl.setUniforms(programInfo, {
    u_matrix: mat,
  });

  // вызывает gl.drawArrays или gl.drawElements
  twgl.drawBufferInfo(gl, bufferInfo);
}
```

и затем давайте вызовем эту функцию, чтобы нарисовать F.

```js
function degToRad(d) {
  return d * Math.PI / 180;
}

const settings = {
  rotation: 150,  // в градусах
};
const fieldOfViewRadians = degToRad(120);

function render() {
  twgl.resizeCanvasToDisplaySize(gl.canvas);

  gl.enable(gl.CULL_FACE);
  gl.enable(gl.DEPTH_TEST);

  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const near = 1;
  const far = 2000;

  // Вычисляем матрицу перспективной проекции
  const perspectiveProjectionMatrix =
      m4.perspective(fieldOfViewRadians, aspect, near, far);

  // Вычисляем матрицу камеры, используя look at.
  const cameraPosition = [0, 0, -75];
  const target = [0, 0, 0];
  const up = [0, 1, 0];
  const cameraMatrix = m4.lookAt(cameraPosition, target, up);

  // поворачиваем F в мировом пространстве
  let worldMatrix = m4.yRotation(degToRad(settings.rotation));
  worldMatrix = m4.xRotate(worldMatrix, degToRad(settings.rotation));
  // центрируем 'F' вокруг его начала
  worldMatrix = m4.translate(worldMatrix, -35, -75, -5);

  // рисуем левый вид
  const left = 0;
  const bottom = 0;
  const width = gl.canvas.width / 2;
  const height = gl.canvas.height;

  gl.viewport(left, bottom, width, height);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем правый вид
  const left2 = width;
  const bottom2 = 0;
  const width2 = width;
  const height2 = height;

  gl.viewport(left2, bottom2, width2, height2);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);
}
render();
```

Это в основном то же самое, что и финальный пример из
[статьи о перспективе](webgl-3d-perspective.html),
за исключением того, что мы используем [нашу библиотеку](webgl-less-code-more-fun.html), чтобы держать код проще.

{{{example url="../webgl-multiple-views-one-view.html"}}}

Теперь давайте сделаем так, чтобы он рисовал 2 вида 'F' бок о бок,
используя `gl.viewport`

```js
function render() {
  twgl.resizeCanvasToDisplaySize(gl.canvas);

  gl.enable(gl.CULL_FACE);
  gl.enable(gl.DEPTH_TEST);

  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const near = 1;
  const far = 2000;

  // Вычисляем матрицу перспективной проекции
  const perspectiveProjectionMatrix =
      m4.perspective(fieldOfViewRadians, aspect, near, far);

  // Вычисляем матрицу камеры, используя look at.
  const cameraPosition = [0, 0, -75];
  const target = [0, 0, 0];
  const up = [0, 1, 0];
  const cameraMatrix = m4.lookAt(cameraPosition, target, up);

  // поворачиваем F в мировом пространстве
  let worldMatrix = m4.yRotation(degToRad(settings.rotation));
  worldMatrix = m4.xRotate(worldMatrix, degToRad(settings.rotation));
  // центрируем 'F' вокруг его начала
  worldMatrix = m4.translate(worldMatrix, -35, -75, -5);

  // рисуем левый вид
  const left = 0;
  const bottom = 0;
  const width = gl.canvas.width / 2;
  const height = gl.canvas.height;

  gl.viewport(left, bottom, width, height);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем правый вид
  const left2 = width;
  const bottom2 = 0;
  const width2 = width;
  const height2 = height;

  gl.viewport(left2, bottom2, width2, height2);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);
}
render();
```

{{{example url="../webgl-multiple-views.html"}}}

Теперь у нас есть 2 вида 'F' бок о бок. Каждый вид использует половину canvas.

Давайте добавим еще один вид, чтобы у нас было 3 вида

```js
function render() {
  twgl.resizeCanvasToDisplaySize(gl.canvas);

  gl.enable(gl.CULL_FACE);
  gl.enable(gl.DEPTH_TEST);

  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const near = 1;
  const far = 2000;

  // Вычисляем матрицу перспективной проекции
  const perspectiveProjectionMatrix =
      m4.perspective(fieldOfViewRadians, aspect, near, far);

  // Вычисляем матрицу камеры, используя look at.
  const cameraPosition = [0, 0, -75];
  const target = [0, 0, 0];
  const up = [0, 1, 0];
  const cameraMatrix = m4.lookAt(cameraPosition, target, up);

  // поворачиваем F в мировом пространстве
  let worldMatrix = m4.yRotation(degToRad(settings.rotation));
  worldMatrix = m4.xRotate(worldMatrix, degToRad(settings.rotation));
  // центрируем 'F' вокруг его начала
  worldMatrix = m4.translate(worldMatrix, -35, -75, -5);

  // рисуем левый вид
  const left = 0;
  const bottom = 0;
  const width = gl.canvas.width / 3;
  const height = gl.canvas.height;

  gl.viewport(left, bottom, width, height);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем средний вид
  const left2 = width;
  const bottom2 = 0;
  const width2 = width;
  const height2 = height;

  gl.viewport(left2, bottom2, width2, height2);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем правый вид
  const left3 = width * 2;
  const bottom3 = 0;
  const width3 = width;
  const height3 = height;

  gl.viewport(left3, bottom3, width3, height3);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);
}
render();
```

{{{example url="../webgl-multiple-views-clear-fixed.html"}}}

Теперь у нас есть 3 вида. Каждый вид использует треть canvas.

Обратите внимание, что каждый вид рисует ту же сцену. Это может быть полезно для отладки или для создания интерфейса с множественными видами.

Давайте также добавим scissor тест, чтобы убедиться, что мы не рисуем за пределами каждого viewport

```js
function render() {
  twgl.resizeCanvasToDisplaySize(gl.canvas);

  gl.enable(gl.CULL_FACE);
  gl.enable(gl.DEPTH_TEST);
  gl.enable(gl.SCISSOR_TEST);

  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const near = 1;
  const far = 2000;

  // Вычисляем матрицу перспективной проекции
  const perspectiveProjectionMatrix =
      m4.perspective(fieldOfViewRadians, aspect, near, far);

  // Вычисляем матрицу камеры, используя look at.
  const cameraPosition = [0, 0, -75];
  const target = [0, 0, 0];
  const up = [0, 1, 0];
  const cameraMatrix = m4.lookAt(cameraPosition, target, up);

  // поворачиваем F в мировом пространстве
  let worldMatrix = m4.yRotation(degToRad(settings.rotation));
  worldMatrix = m4.xRotate(worldMatrix, degToRad(settings.rotation));
  // центрируем 'F' вокруг его начала
  worldMatrix = m4.translate(worldMatrix, -35, -75, -5);

  // рисуем левый вид
  const left = 0;
  const bottom = 0;
  const width = gl.canvas.width / 3;
  const height = gl.canvas.height;

  gl.viewport(left, bottom, width, height);
  gl.scissor(left, bottom, width, height);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем средний вид
  const left2 = width;
  const bottom2 = 0;
  const width2 = width;
  const height2 = height;

  gl.viewport(left2, bottom2, width2, height2);
  gl.scissor(left2, bottom2, width2, height2);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем правый вид
  const left3 = width * 2;
  const bottom3 = 0;
  const width3 = width;
  const height3 = height;

  gl.viewport(left3, bottom3, width3, height3);
  gl.scissor(left3, bottom3, width3, height3);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);
}
render();
```

{{{example url="../webgl-multiple-views-clear-issue.html"}}}

Теперь у нас есть scissor тест, который гарантирует, что мы не рисуем за пределами каждого viewport.

Давайте также добавим очистку для каждого viewport

```js
function render() {
  twgl.resizeCanvasToDisplaySize(gl.canvas);

  gl.enable(gl.CULL_FACE);
  gl.enable(gl.DEPTH_TEST);
  gl.enable(gl.SCISSOR_TEST);

  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const near = 1;
  const far = 2000;

  // Вычисляем матрицу перспективной проекции
  const perspectiveProjectionMatrix =
      m4.perspective(fieldOfViewRadians, aspect, near, far);

  // Вычисляем матрицу камеры, используя look at.
  const cameraPosition = [0, 0, -75];
  const target = [0, 0, 0];
  const up = [0, 1, 0];
  const cameraMatrix = m4.lookAt(cameraPosition, target, up);

  // поворачиваем F в мировом пространстве
  let worldMatrix = m4.yRotation(degToRad(settings.rotation));
  worldMatrix = m4.xRotate(worldMatrix, degToRad(settings.rotation));
  // центрируем 'F' вокруг его начала
  worldMatrix = m4.translate(worldMatrix, -35, -75, -5);

  // рисуем левый вид
  const left = 0;
  const bottom = 0;
  const width = gl.canvas.width / 3;
  const height = gl.canvas.height;

  gl.viewport(left, bottom, width, height);
  gl.scissor(left, bottom, width, height);
  gl.clearColor(0.2, 0.2, 0.2, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем средний вид
  const left2 = width;
  const bottom2 = 0;
  const width2 = width;
  const height2 = height;

  gl.viewport(left2, bottom2, width2, height2);
  gl.scissor(left2, bottom2, width2, height2);
  gl.clearColor(0.2, 0.2, 0.2, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем правый вид
  const left3 = width * 2;
  const bottom3 = 0;
  const width3 = width;
  const height3 = height;

  gl.viewport(left3, bottom3, width3, height3);
  gl.scissor(left3, bottom3, width3, height3);
  gl.clearColor(0.2, 0.2, 0.2, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);
}
render();
```

{{{example url="../webgl-multiple-views-clear-fixed.html"}}}

Теперь у нас есть очистка для каждого viewport, что гарантирует, что каждый вид имеет чистый фон.

Давайте также добавим возможность рисовать разные сцены в каждом viewport

```js
function render() {
  twgl.resizeCanvasToDisplaySize(gl.canvas);

  gl.enable(gl.CULL_FACE);
  gl.enable(gl.DEPTH_TEST);
  gl.enable(gl.SCISSOR_TEST);

  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const near = 1;
  const far = 2000;

  // Вычисляем матрицу перспективной проекции
  const perspectiveProjectionMatrix =
      m4.perspective(fieldOfViewRadians, aspect, near, far);

  // Вычисляем матрицу камеры, используя look at.
  const cameraPosition = [0, 0, -75];
  const target = [0, 0, 0];
  const up = [0, 1, 0];
  const cameraMatrix = m4.lookAt(cameraPosition, target, up);

  // поворачиваем F в мировом пространстве
  let worldMatrix = m4.yRotation(degToRad(settings.rotation));
  worldMatrix = m4.xRotate(worldMatrix, degToRad(settings.rotation));
  // центрируем 'F' вокруг его начала
  worldMatrix = m4.translate(worldMatrix, -35, -75, -5);

  // рисуем левый вид
  const left = 0;
  const bottom = 0;
  const width = gl.canvas.width / 3;
  const height = gl.canvas.height;

  gl.viewport(left, bottom, width, height);
  gl.scissor(left, bottom, width, height);
  gl.clearColor(0.2, 0.2, 0.2, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
  drawScene(perspectiveProjectionMatrix, cameraMatrix, worldMatrix);

  // рисуем средний вид с другой камерой
  const left2 = width;
  const bottom2 = 0;
  const width2 = width;
  const height2 = height;

  gl.viewport(left2, bottom2, width2, height2);
  gl.scissor(left2, bottom2, width2, height2);
  gl.clearColor(0.2, 0.2, 0.2, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  // используем другую камеру для среднего вида
  const cameraPosition2 = [0, 0, -50];
  const cameraMatrix2 = m4.lookAt(cameraPosition2, target, up);
  drawScene(perspectiveProjectionMatrix, cameraMatrix2, worldMatrix);

  // рисуем правый вид с другой камерой
  const left3 = width * 2;
  const bottom3 = 0;
  const width3 = width;
  const height3 = height;

  gl.viewport(left3, bottom3, width3, height3);
  gl.scissor(left3, bottom3, width3, height3);
  gl.clearColor(0.2, 0.2, 0.2, 1);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  // используем другую камеру для правого вида
  const cameraPosition3 = [0, 0, -100];
  const cameraMatrix3 = m4.lookAt(cameraPosition3, target, up);
  drawScene(perspectiveProjectionMatrix, cameraMatrix3, worldMatrix);
}
render();
```

{{{example url="../webgl-multiple-views-items.html"}}}

Теперь у нас есть 3 вида с разными камерами. Каждый вид показывает ту же сцену с разных ракурсов.

Конечно, вы могли бы рисовать целые 3D сцены или что угодно для каждого элемента.
Пока вы правильно устанавливаете viewport и scissor, а затем настраиваете
вашу матрицу проекции, чтобы соответствовать аспекту области, это должно работать.

Еще одна примечательная вещь о коде - мы перемещаем canvas
с этой строкой

```
gl.canvas.style.transform = `translateY(${window.scrollY}px)`;
```

Почему? Мы могли бы вместо этого установить canvas в `position: fixed;`, в этом случае
он не прокручивался бы со страницей. Разница была бы тонкой.
Браузер пытается прокручивать страницу как можно более плавно. Это может быть
быстрее, чем мы можем рисовать наши объекты. Из-за этого у нас есть 2 варианта.

1. Использовать canvas с фиксированной позицией

   В этом случае, если мы не можем обновляться достаточно быстро, HTML перед canvas будет прокручиваться, но сам canvas
   не будет, поэтому на несколько мгновений они будут не синхронизированы

   <img src="resources/multi-view-skew.gif" style="border: 1px solid black; width: 266px;" class="webgl_center">

2. Перемещать canvas под контентом

   В этом случае, если мы не можем обновляться достаточно быстро, canvas будет прокручиваться в синхронизации
   с HTML, но новые области, где мы хотим рисовать вещи, будут пустыми, пока мы не получим
   шанс нарисовать.

   <img src="resources/multi-view-fixed.gif" style="border: 1px solid black; width: 266px;" class="webgl_center">

   Это решение, используемое выше

Надеюсь, эта статья дала вам некоторые идеи о том, как рисовать множественные виды.
Мы будем использовать эти техники в нескольких будущих статьях, где
возможность видеть множественные виды полезна для понимания.

<div class="webgl_bottombar" id="pixel-coords">
<h3>Координаты пикселей</h3>
<p>Координаты пикселей в WebGL
ссылаются на их края. Так, например, если у нас был
canvas размером 3x2 пикселя, и мы установили viewport
как</p>
<pre class="prettyprint"><code>
gl.viewport(
  0, // left
  0, // bottom
  3, // width
  2, // height
);
</code></pre>
<p>Тогда мы действительно определяем этот прямоугольник, который окружает 3x2 пикселя</p>
<div class="webgl_center"><img src="resources/webgl-pixels.svg" style="width: 500px;"></div>
<p>Это означает, что значение пространства отсечения X = -1.0 соответствует левому краю этого прямоугольника,
а значение пространства отсечения X = 1.0 соответствует правому. Выше я сказал, что X = -1.0 соответствует самому левому пикселю,
но на самом деле соответствует левому краю</p>
</div>
