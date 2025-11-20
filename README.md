# Tarea S9 IG - Leslie Liu Romero Martín

## Link de CodeSandbox

- https://codesandbox.io/p/sandbox/ig-tarea-s9-entrega-basada-en-s8-nk6h2w

## Introducción

En esta tarea de shaders, me decidí por la opción de implementar shaders en una práctica anterior, en este caso la práctica de la semana pasada (S8 - Representación de volcanes). Desde un principio, decidí que para los volcanes había que añadir algún efecto que ayudase a la ambientación cada vez que se hacía click en uno de los marcadores.

En el proceso de decisión, inspirándome en los ejemplos vistos en clase, me decidí por modificar la textura de la tierra cuando hayamos hecho click en un marcador de volcán (inspirado por doble textura y la idea de aplicar un filtro) y añadir algún tipo de efecto a la acción de hacer clic que se asimilara a una onda (ejemplo de GitHub de círculos concéntricos). Aunque con respecto a la implementación hay pocas similitudes, la idea surgió de ahí.

## Shaders para la textura de la Tierra

Para la textura de la Tierra, como mencioné anteriormente, añadí el efecto de que la textura pasase de su estado normal a un color que recuerda a un volcán, con tonos rojizos y negros, esto se hace progresivamente con la ayuda de una variable uniform (`uVolcanic`), cuyo valor se empieza a incrementar cuando hacemos click en un marcador de volcan, creando una sensación de que la Tierra se oscurece progresivamente hasta alcanzar el tono rojizo mencionado anteriormente. Al pulsar fuera, la variable se decrementa nuevamente hasta llegar a cero, lo que devuelve la textura de la Tierra a su estado original.

Para su implementación, ya que se trata de un efecto del color, usamos el Fragment Shader. A continuación se encuentra la función `volcanicEarthFragment`:

```js
function volcanicEarthFragment() {
  return `
    precision mediump float;
    uniform sampler2D uTexture;
    uniform float uVolcanic;
    varying vec2 vUv;
    
    vec3 volcanicGradient(float x) {
        vec3 black = vec3(0.0, 0.0, 0.0);
        vec3 darkred = vec3(0.3, 0.0, 0.0);
        vec3 lava = vec3(1.0, 0.1, 0.0);
    
        return mix(
            mix(black, darkred, smoothstep(0.0, 0.5, x)),
            lava,
            smoothstep(0.5, 1.0, x)
        );
    }
    
    void main() {
        vec4 tex = texture2D(uTexture, vUv);
        float g = dot(tex.rgb, vec3(0.299, 0.587, 0.114));
        vec3 volcanic = volcanicGradient(g);
        vec3 finalColor = mix(tex.rgb, volcanic, uVolcanic);
        gl_FragColor = vec4(finalColor, 1.0);
    }
  
  `
}
```

Como se puede ver, en su mayoría se trata de combinar los colores característicos de volcanes (rojo oscuro, negro y rojo intenso) y combinarlos con la textura de la Tierra que se pasa a través de la variable uniform `uTexture` para formar la versión rojiza de la Tierra.

El Vertex Fragment no tiene ningún añadido:

```js
function volcanicEarthVertex() {
  return `
    varying vec2 vUv;
    void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `
}
```

Como nota, debo comentar que, al tener que cambiar el material de la esfera de la Tierra de MeshPhongMaterial a ShaderMaterial, los efectos del BumpMap y el SpecularMap se pierden, así como los otros efectos de luces que se encontraban presentes en la entrega anterior, ya que por sencillez no implementé toda la funcionalidad de luces y sombras en el shader.

![GIF de la textura "volcánica" en la Tierra](examples/volcanic_texture.gif)

## Las ondas al pulsar

En un principio, mi idea era implementar algo similar al último ejemplo que pudimos ver en la sesión práctica, es decir, la onda en el eje Z que realizaba un movimiento sinosuidal. Empecé con una capa por encima de la Tierra que se mostraba en rojo con transparencia e intenté generar ondas centradas en el click, sin embargo, al probar varias implementaciones, no me convencieron los resultados, por lo que decidí realizar algo más sencillo. Al final, decidí implementar unas ondas que utilizaran el Fragment Shader en vez del Vertex Shader y por tanto usaran el color para simular el efecto de ondas expandiéndose desde el marcador en el que se ha hecho click.

De esta manera, el Vertex Shader vuelve a ser trivial:

```js
function vertexShader() {
  return `
    varying vec2 vUv;
    varying vec3 vPos;
    
    void main() {
      vUv = uv;
      vPos = position;

      gl_Position = projectionMatrix *
                    modelViewMatrix *
                    vec4(position, 1.0);
    }
  `;
}
```

Y en el Fragment Shader nuevamente se realiza toda la implementación:

```js
function fragmentShader() {
  return `
  uniform float uTime;
  uniform float uRippleTime;
  uniform vec3 uCenter;
  varying vec2 vUv;
  varying vec3 vPos;

  void main() {

      float timeSince = uTime - uRippleTime;

      vec4 finalColor = vec4(0.0, 0.0, 0.0, 0.0);

      if (uRippleTime > 0.0 && timeSince < 30.0) {
        float d = distance(normalize(vPos), normalize(uCenter));

        float speed = 2.0;
        float wavelength = 0.05;
        float ringWidth = 0.05;
    
        float waveInput = d / wavelength - timeSince / speed;

        float ringPattern = fract(waveInput);

        float ring = 1.0 - smoothstep(0.0, ringWidth, ringPattern);
        float falloff = exp(-timeSince * 1.0);
        float alpha = ring * falloff;

        vec3 rippleColor = vec3(1.0, 0.5, 0.0);

        finalColor = vec4(rippleColor, alpha);

      }

      gl_FragColor = finalColor;
  }

  
  `;
}
```

Los uniforms representan, respectivamente, el tiempo que ha transcurrido (`uTime`), el timestamp del momento en el que se hace click (se utiliza luego para calcular el tiempo durante el que se va a mostrar el efecto; `uTimeRipple`) y el punto en el que se realizado el click, el cual será el centro de las esferas concéntricas u olas que se generan (`uCenter`).

Como se puede observar, se calcula el tiempo que ha pasado desde el click, lo cual se utilizará luego para la atenuación de las ondas. Luego, con una duración de 30s desde que se realiza el click, se calcula la distancia (`d`) al centro en coordenadas de la esfera y se establecen los parámetros para las ondas (velocidad, frecuencia, anchura del anillo). Posteriormente, se calcula el efecto de las ondas generándose hacia fuera en círculos concéntricos teniendo en cuenta la componente espacial (`d/wavelength`) y la componente temporal (`timeSince/speed`). A partir del cálculo anterior, generamos el patrón repetitivo de las ondas, que luego se traducirán a las líneas reales visibles. Al final, obtenemos el conjunto completo al tener en cuenta la desaparición de los anillos con el paso del tiempo (`falloff`).

![GIF de los anillos al pulsar un marcador](examples/anillos.gif)

## Últimas observaciones

Como última nota, queria comentar que añadí la parte faltante de audio que quise implementar en la práctica anterior pero para lo que no tuve tiempo, sin embargo, parece no funcionar de ninguna manera. Quería comentarlo por dejar constancia de que está implementado, aunque sé que no es parte de esta práctica, y por si fuese un problema de mi ordenador el hecho de que no se escuche. Brevemente, la funcionalidad del audio es que, al pulsar una esfera, junto con los efectos visuales, debería escucharse de fondo un sonido de lava.

```js
function init(){

    // [...]

    const listener = new THREE.AudioListener();
    camera.add( listener );
    const audioLoader = new THREE.AudioLoader();

    // [...]

    audioLoader.load( 
        '/src/sound1.mp3',
        function ( buffer ) {
            console.log("Audio buffer loaded successfully:", buffer);

            for (let marker of Markers) {
                const markerSound = new THREE.PositionalAudio( listener );
                
                markerSound.setBuffer( buffer );
                markerSound.setLoop( false );
                markerSound.setVolume( 1.0 );
                markerSound.setRefDistance( 100 );
                
                marker.add( markerSound );
            }
        }
    );

    function onClick(event) {
        // [...]

        if (intersects.length > 0) {
            const marker = intersects[0].object;

            const soundToPlay = marker.children.find(child => child.isPositionalAudio);     // <--

            if (marker.userData) {
                // ... acciones con los datos al hacer click ...

            }

            heatUp = true;
            uniforms.uRippleTime.value = uniforms.uTime.value;
            const worldPos = new THREE.Vector3();
            marker.getWorldPosition(worldPos);
            uniforms.uCenter.value.copy(worldPos);

            // Sonido de lava
            if (soundToPlay && !soundToPlay.isPlaying) {
                soundToPlay.play();
            }
        } else {
            // ... resto de la lógica ...
        }

    }
}
```
En el vídeo final, el cual se encuentra en este repositorio y del que dejo un recorte a continuación, se escucha de fondo la pista de audio que debería escucharse al pulsar en un marcador (en el vídeo lo puse yo manual de manera externa al código).

![Parte del Vídeo final](examples/video_final.gif)

## Referencias

- The Book of Shaders: https://thebookofshaders.com/

NOTA: Para la realización de esta práctica, me he ayudado de la IA generativa.
