<script>
  import { onMount } from "svelte";
  import * as d3 from "d3"
  import albums from "/src/data/albums_copy.csv"

  let streams = [];
  let anios = [];
  let anios_str = [];
  let currentYear = 2003; // Año inicial
  let maxYear = 2023; // Año máximo

  for (let i = 0; i < albums.length; i++) {
    streams.push(parseInt(albums[i].Streams));
  }

  for (let i = 0; i < 21; i++){
    anios.push(2003+i);
    anios_str.push((2003+i).toString());
  }

  const color_genero = {
    "Hip Hop/Rap": ["195",	"45",	"107", "0.75"],
    "Pop": ["210",	"111",	"235", "0.75"],
    "Rock": ["210",	"88",	"11", "0.75"],
    "R&B/Soul": ["1",	"151",	"246", "0.75"],
    "Latin/Regueton": ["252",	"186",	"4", "0.75"],
    "Otros": ["95",	"205",	"138", "0.75"],
  }

  let bubble_size = d3
    .scaleRadial()
    .domain([d3.min(streams), d3.max(streams)])
    .range([85, 200])

  function albums_of(anio){
    const begin = (anio - 2003)*5;
    const end = begin + 5;
    const rta = albums.slice(begin, end);
    return rta;
  }

  function genres(generos){
    const rta = generos.split(";");
    return rta;
  }

  function generarPosiciones(anio) {
    let rv = [];
    const ancho = window.innerWidth;
    const alto = window.innerHeight;
    const curr_albums = albums_of(anio);
    let sizes = [];
    for (let i = 0; i < 5; i++) {
      sizes.push(bubble_size(parseInt(curr_albums[i].Streams)));
    }
    const cubiculos = [[ancho*0.1,alto*0.15], [ancho*0.75,alto*0.25], [ancho*0.45,alto*0.35], [ancho*0.25,alto*0.60], [ancho*0.6,alto*0.65]];
    for (var i = 0; i < 5; i++) {
      let x = d3.randomUniform(-25,25)() + cubiculos[i][0];
      let y = d3.randomUniform(-25,25)() + cubiculos[i][1];
      while (y + sizes[i] + 120 > alto){
        y -= 25;
      }
      rv.push([x, y]);
    }
    return rv;
  }

  function isOverlapping(newBubble, main_bubbles) {
    const A = newBubble.x;
    const B = newBubble.y;
    const C = newBubble.x + newBubble.radius * 2;
    const D = newBubble.y + newBubble.radius * 2; 
    return main_bubbles.some(mainBubble => {
      const rect = mainBubble.getBoundingClientRect();
      const rect_text = mainBubble.querySelector("p").getBoundingClientRect();
      const a = rect.left-50;
      const b = (rect.top % window.innerHeight)-50;
      const c = a + rect.width+100;
      const d = b + rect.height+rect_text.height+100; 

      const superopone = (
        (a<A && A<c && b<B && B<d) ||
        (a<A && A<c && b<D && D<d) ||
        (a<C && C<c && b<B && B<d) ||
        (a<C && C<c && b<D && D<d)
      );

      return superopone;
    });
  }

  function generateFillBubbles(main_bubbles) {
    const fillBubbles = [];
    const numFillBubbles = parseInt(d3.randomUniform(50,70)(1)); // Número de burbujas de relleno
    for (let i = 0; i < numFillBubbles; i++) {
      const radius = parseInt(d3.randomUniform(10,40)(1)); // Tamaño aleatorio
      //const x = Math.random() * (window.innerWidth - radius * 2) + radius;
      const x = d3.randomUniform(-50, window.innerWidth-50)(1);
      const y = d3.randomUniform(-50, window.innerHeight)(1);
      //const y = Math.random() * (window.innerHeight - radius * 2) + radius;
      const newBubble = {"x": x, "y": y, "radius": radius};
      if (!isOverlapping(newBubble, main_bubbles)) {
        fillBubbles.push(newBubble);
      } else {
        i--; // Si hay superposición, intenta nuevamente
      }
    }
    return fillBubbles;
  }

  function posicion_circulos(n) {
    const vertices = [];
    const r = 25;
    // Caso n = 1
    if (n === 1) {
      return [[0, 0]]; // Única posición en el centro
    }

    for (let k = 0; k < n; k++) {
      const theta = ((2 * Math.PI * k) / n) + Math.PI/2;
      const pos = [r * Math.cos(theta), r * Math.sin(theta)];
      vertices.push(pos);
    }
    
    return vertices;
  }

  function recuadro_valoracion(valoracion){
    if (valoracion == 1){
      return "/images/circle.png";
    }
    if (valoracion == 2){
      return "/images/dashed_circle.png";
    }
    if (valoracion == 4){
      return "/images/picos_circle.png";
    }
  }
  
  let posiciones = [];
  for (let i = 0; i < 21; i++) {
    posiciones.push(generarPosiciones(2003+i));
  }

  function posiciones_for(anio){
    return posiciones[anio-2003];
  }

  const observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add("show");
      } else {
        entry.target.classList.remove("show");
      }
    });
  },
  {
    threshold: 0.5
  }
  );

  const handleWheel = (event) => {
    if (!isScrolling) {
      isScrolling = true;
      container.removeEventListener('wheel', handleWheel);
      if (event.deltaY > 0) {
        // Scroll hacia abajo
        container.scrollBy({
          top: window.innerHeight, // Desplazarse hacia abajo una página
          behavior: 'smooth'
        });
      } else {
        // Scroll hacia arriba
        container.scrollBy({
          top: -window.innerHeight, // Desplazarse hacia arriba una página
          behavior: 'smooth'
        });
      }
      // Restablecer el estado después de un tiempo
      setTimeout(() => {
        isScrolling = false;
        container.addEventListener('wheel', handleWheel);
      }, 0); // Ajusta el tiempo según sea necesario
    }
    event.preventDefault(); // Prevenir el comportamiento predeterminado
  };


  let fill_bubbles = [];
  document.addEventListener('DOMContentLoaded', () => {
    const targets = document.querySelectorAll(".page");
    targets.forEach((el) => {
      observer.observe(el);
    });
    
    const container = document.querySelector('.container');
    var isScrolling = false;   
    //container.addEventListener('wheel', handleWheel); 

    const paginas = document.querySelectorAll('.vis');
    for (let i = 0; i < paginas.length; i++) {
      const filling = document.createElement('div');
      filling.className = 'filling';
      let mainBubbles = Array.from(paginas[i].querySelectorAll(".album_container"));
      fill_bubbles = generateFillBubbles(mainBubbles);
      fill_bubbles.forEach(bubble => {
        const bubbleElement = document.createElement('div');
        bubbleElement.className = 'bubble_filler';
        bubbleElement.style.left = `${bubble.x}px`;
        bubbleElement.style.top = `${bubble.y}px`;
        bubbleElement.style.animationDelay = `${Math.random() * 3}s`;
        bubbleElement.style.animationDuration = `${4 + Math.random() * 2}s`;
        bubbleElement.style.zIndex = -1;
        bubbleElement.style.opacity = d3.min([1, d3.randomUniform(0.1,0.5)(1)+bubble.radius/100]);
        const imagen = document.createElement('img');
        imagen.src = 'images/burbuja.png';
        imagen.alt = 'Bubble';
        imagen.style.position = 'absolute';
        imagen.style.maxWidth = `${bubble.radius*2}px`;
        imagen.style.height = 'auto';
        imagen.position = 'absolute';
        bubbleElement.appendChild(imagen);
        filling.appendChild(bubbleElement);
      });
      paginas[i].appendChild(filling);
    }
  });

</script>

<main>
  <div class="container">
    <div class="page intro">
      <h1>Musical Bubbles</h1>
    </div>
    <div class="page intro">
      <h1>cheat sheet</h1>
    </div>
    {#each anios as anio,index}
      <div class="page vis">
        <div class="info">
          {#each albums_of(anio) as album, index}
            <div class="album_container" style="width: {bubble_size(parseInt(album.Streams))}px; height: {bubble_size(parseInt(album.Streams))}px; top: {(posiciones_for(anio))[index][1]}px; left: {(posiciones_for(anio))[index][0]}px; animation-delay: {Math.random() * 3}s; animation-duration: {4 + Math.random() * 2}s;">      
              <div class="bubble">
                {#each posicion_circulos((genres(album.Generos).length)) as [x, y], index}
                  {#if genres(album.Generos).length != 1}
                    <div class="circle" style="left: {x}%; top: {y}% ; width: 75%; height: 75%; background-color: rgba({color_genero[genres(album.Generos)[index]][0]},{color_genero[genres(album.Generos)[index]][1]},{color_genero[genres(album.Generos)[index]][2]},{color_genero[genres(album.Generos)[index]][3]});"></div>
                  {/if}
                  {#if genres(album.Generos).length == 1}  
                    <div class="circle" style="transform: translate(0%, -5%); width: 110%; height: 110%; background-color: rgba({color_genero[genres(album.Generos)[index]][0]},{color_genero[genres(album.Generos)[index]][1]},{color_genero[genres(album.Generos)[index]][2]},{color_genero[genres(album.Generos)[index]][3]});"></div>
                  {/if}
                {/each}
                <a style="z-index:3" href="https://www.google.com/?hl=es">
                  <img src="images/burbuja.png" alt="Bubble" style="width: {bubble_size(parseInt(album.Streams))}px; height: {bubble_size(parseInt(album.Streams))}px">
                </a>
                {#if album.Valoracion == 1}  
                  <img src="/images/circle.png" alt="Circle" style="position: absolute; transform: translate(0%, -5.5%);max-width: {bubble_size(parseInt(album.Streams))*1.125}px; max-height: {bubble_size(parseInt(album.Streams))*1.125}px">
                {/if}
                {#if album.Valoracion == 2}  
                  <img src="/images/dashed_circle.png" alt="Circle" style="position: absolute; transform: translate(0%, -5.5%);max-width: {bubble_size(parseInt(album.Streams))*1.125}px; max-height: {bubble_size(parseInt(album.Streams))*1.125}px">
                {/if}
                {#if album.Valoracion == 4}  
                  <img src="/images/picos_circle.png" alt="Circle" style="position: absolute; transform: translate(0%, -5.5%);max-width: {bubble_size(parseInt(album.Streams))*1.125}px; max-height: {bubble_size(parseInt(album.Streams))*1.125}px">
                {/if}
                {#if album.aoty == 1}  
                  <div class="duck">
                    <img src="/images/patito.png" alt="Duck" style="max-width: {bubble_size(parseInt(album.Streams))*0.25}px; max-height: {bubble_size(parseInt(album.Streams))*0.25}px">
                  </div>
                {/if}
              </div>
              <p>{album["Album"]} <br><span>{album["Artista"]}</span></p>
            </div>
          {/each}
        </div>
        <div class = "anio">
          {#each anios_str[index] as caracter}
            <div class="caracter" style="animation: {4+Math.random()*2}s float ease-in-out infinite; animation-delay:{Math.random() * 3}s">
              <h1>{caracter}</h1>
            </div>
          {/each}
        </div>
        <div class="help">
          <img src="/images/boton_help_blue.png" alt="help" style="width: 35px; height:35px">          
          <div class="help_cuadro" style="width: 120vh; height:100%">
            <img src="/images/cheat guide.png" alt="mini cheat_sheet" style="border-radius: 10px;border: 2px solid white;">
          </div>
        </div>
      </div>
    {/each}
  </div>
</main>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Karla:ital,wght@0,200..800;1,200..800&family=Nunito:ital,wght@0,200..1000;1,200..1000&display=swap');

  .container {
    scroll-snap-type: y mandatory;
    overflow: hidden;
    overflow-y: scroll;
    width: 100%;
    height: 100vh;
  }
  
  .page {
    scroll-snap-align: start;
    padding: 2rem;
    width: 100%;
    height: 100%;
    display: flex;
    opacity: 0;
    transition: all 1s ease-in-out;
  }

  .intro {
    align-items: center;
    justify-content: center;
  }

  .info {
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    max-width: 100%;
    max-height: 100%;
  }

  .album_container {
    position: absolute;
    width: auto;
    height: auto;
    animation: float ease-in-out infinite;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
  }

  .bubble {
    position: relative;
    width: auto;
    height: auto;
    display: flex;
    justify-content: center;
  }

  @keyframes bounce2 {
    0%, 25%, 80%, 100% {
      transform: translateY(0) rotateZ(0deg); /* Posición inicial, sin rotación */
    }
    40% {
      transform: translateY(-150%) rotateZ(-360deg); /* Salto hacia arriba y rotación hacia atrás */
    }
    52% {
      transform: rotateZ(0deg); /* Salto hacia arriba y rotación hacia atrás */
    }
    60% {
      transform: translateY(-7.5px) ; /* Comienza a bajar, con rotación hacia atrás */
    }
  }

  .duck {
    animation: bounce2 2s infinite;
    position: absolute;
    z-index: 2;
    top: -20%;
  }

  .bubble img {
    position: relative;
    z-index: 1;
  }

  .circle {
    position: absolute;
    border-radius: 50%;
    filter: blur(10px);
    transform: translate(15%, 15%);
    z-index: 0;
  }

  .album_container p {
    font-family: "Karla", sans-serif;
    font-optical-sizing: auto;
    font-weight: 400;
    font-style: normal;
    color: white;
    margin-top: 20px;
    font-size: 16px;
    position: absolute;
    top: 95%;
    z-index: 1;
  }

  .album_container span {
    font-family: "Karla", sans-serif;
    font-optical-sizing: auto;
    font-weight: 400;
    font-style: normal;
    font-size: 14px;
    color: #ebeaea;
  }

  .anio {
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    top: 85%;
    left: 85%;
    max-width: 100px;
    max-height: 100px;
  }

  @font-face {
    font-family: 'Bubbly';
    src: url('/fonts/ShineBubbleDeco-MVKJx.ttf') format('truetype');
    font-weight: normal;
    font-style: normal;
  }

  h1 {
    font-family: 'Bubbly', sans-serif;
    font-size: 3rem;
    color: white;
  }

  .help {
    position:relative; 
    left: 87.5%;
  } 

  .help_cuadro {
    position:absolute;
    transform: translateX(-100%);
    visibility: hidden;
    max-height: 100%;
    z-index: 3;
  }

  .help:hover .help_cuadro {
    visibility: visible;
  }

</style>
