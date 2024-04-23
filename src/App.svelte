<script>
  import { onMount } from "svelte";
  import axios from "axios";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

  let magFilter = 20; // Initialwert für den Filter
  // $: magFilterInverted = 20 - magFilter;
  let scene, camera, renderer;
  let showProperStarsOnly = false;
  let sonneSichtbar = true;
  let sonneMesh;
  let stars = [];

  const hauptSternzeichen = [
    "Ari",
    "Tau",
    "Gem",
    "Cnc",
    "Leo",
    "Vir",
    "Lib",
    "Sco",
    "Sgr",
    "Cap",
    "Aqr",
    "Psc",
  ];

  // Alle Konstellationen aus deiner Frage
  let alleKonstellationen = [
    { con: "And" },
    { con: "Ant" },
    { con: "And" },
    { con: "Ant" },
    { con: "Aps" },
    { con: "Aql" },
    { con: "Aqr" },
    { con: "Ara" },
    { con: "Ari" },
    { con: "Aur" },
    { con: "Boo" },
    { con: "CMa" },
    { con: "CMi" },
    { con: "CVn" },
    { con: "Cae" },
    { con: "Cam" },
    { con: "Cap" },
    { con: "Car" },
    { con: "Cas" },
    { con: "Cen" },
    { con: "Cep" },
    { con: "Cet" },
    { con: "Cha" },
    { con: "Cir" },
    { con: "Cnc" },
    { con: "Col" },
    { con: "Com" },
    { con: "CrA" },
    { con: "CrB" },
    { con: "Crt" },
    { con: "Cru" },
    { con: "Crv" },
    { con: "Cyg" },
    { con: "Del" },
    { con: "Dor" },
    { con: "Dra" },
    { con: "Eco" },
    { con: "Equ" },
    { con: "Eri" },
    { con: "For" },
    { con: "Gem" },
    { con: "Gru" },
    { con: "Her" },
    { con: "Hor" },
    { con: "Hya" },
    { con: "Hyi" },
    { con: "Ind" },
    { con: "LMi" },
    { con: "Lac" },
    { con: "Leo" },
    { con: "Lep" },
    { con: "Lib" },
    { con: "Lup" },
    { con: "Lyn" },
    { con: "Lyr" },
    { con: "Men" },
    { con: "Mic" },
    { con: "Mon" },
    { con: "Mus" },
    { con: "Nor" },
    { con: "Oct" },
    { con: "Oph" },
    { con: "Ori" },
    { con: "Pav" },
    { con: "Peg" },
    { con: "Per" },
    { con: "Phe" },
    { con: "Pic" },
    { con: "PsA" },
    { con: "Psc" },
    { con: "Pup" },
    { con: "Pyx" },
    { con: "Ret" },
    { con: "Scl" },
    { con: "Sco" },
    { con: "Sct" },
    { con: "Ser" },
    { con: "Sex" },
    { con: "Sge" },
    { con: "Sgr" },
    { con: "Sir" },
    { con: "Tau" },
    { con: "Tel" },
    { con: "TrA" },
    { con: "Tri" },
    { con: "Tuc" },
    { con: "UMa" },
    { con: "UMi" },
    { con: "Vel" },
    { con: "Vir" },
    { con: "Vol" },
    { con: "Vul" },
  ];

  // Trenne die Konstellationen in Hauptsternzeichen und andere
  let hauptKonstellationen = alleKonstellationen.filter((konstellation) =>
    hauptSternzeichen.includes(konstellation.con)
  );
  let andereKonstellationen = alleKonstellationen.filter(
    (konstellation) => !hauptSternzeichen.includes(konstellation.con)
  );

  let ausgewaehlteKonstellation = "";
  $: {
    if (ausgewaehlteKonstellation) {
      loadStarsForConstellation(ausgewaehlteKonstellation);
    }
  }

  async function loadStarsForConstellation(constellation) {
    try {
      const response = await axios.get(
        `http://127.0.0.1:3001/stars/${constellation}`
      );

      // Entferne die aktuellen Sterne aus der Szene
      stars.forEach((star) => {
        if (star.mesh) scene.remove(star.mesh);
      });

      // Aktualisiere das 'stars'-Array mit den neuen Daten
      stars = response.data.map((star) => ({
        ...star,
        x: star.x0,
        y: star.y0,
        z: star.z0,
        mesh: null, // Initialisiere 'mesh' als null
      }));

      // Füge neue Sterne basierend auf den geladenen Daten hinzu
      stars.forEach((data) => {
        const size = getSizeByAbsMag(data.absmag); // Berechne die Größe basierend auf dem 'mag'-Wert
        const color = getColorByCI(data.ci); // Bestimme die Farbe basierend auf dem CI-Wert
        const intensity = getIntensityByMag(data.mag); // Berechne die Intensität basierend auf dem 'mag'-Wert

        // Erstelle das Material mit einer Emissionsfarbe basierend auf der Intensität
        const material = new THREE.MeshStandardMaterial({
          color: color,
          emissive: color,
          emissiveIntensity: intensity,
        });

        // Erstelle die Geometrie und das Mesh wie zuvor
        const geometry = new THREE.SphereGeometry(size, 32, 32);
        const sphere = new THREE.Mesh(geometry, material);
        sphere.position.set(data.x, data.y, data.z);
        scene.add(sphere);
        data.mesh = sphere;
      });

      // Aktualisiere die Sichtbarkeit der Sterne basierend auf den aktuellen Filtern
      updateStarVisibility();
    } catch (error) {
      console.error("Fehler beim Laden der Sterndaten: ", error);
    }
  }

  onMount(async () => {
    // Szene, Kamera und Renderer initialisieren
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      10000
    );
    renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    // Sterne laden und hinzufügen
    stars = await loadStars();
    console.log(stars.length);
    const magValues = stars.map((star) => star.mag);

    // Verwende Math.min() und Math.max() mit dem Spread-Operator, um den niedrigsten und höchsten Wert zu finden
    const minMag = Math.min(...magValues);
    const maxMag = Math.max(...magValues);

    console.log(`Niedrigster mag-Wert: ${minMag}`);
    console.log(`Höchster mag-Wert: ${maxMag}`);

    const sunGeometry = new THREE.SphereGeometry(5, 32, 32); // Radius von 5, aber du kannst dies anpassen
    const sunMaterial = new THREE.MeshBasicMaterial({ color: 0xffff00 }); // Gelbe Farbe für die Sonne
    sonneMesh = new THREE.Mesh(sunGeometry, sunMaterial);
    sonneMesh.position.set(0.000005, 0, 0); // Setze die Position der Sonne
    scene.add(sonneMesh);

    stars.forEach((data) => {
      const size = getSizeByAbsMag(data.absmag); // Berechne die Größe basierend auf dem 'mag'-Wert
      const color = getColorByCI(data.ci); // Bestimme die Farbe basierend auf dem CI-Wert
      const intensity = getIntensityByMag(data.mag); // Berechne die Intensität basierend auf dem 'mag'-Wert

      // Erstelle das Material mit einer Emissionsfarbe basierend auf der Intensität
      const material = new THREE.MeshStandardMaterial({
        color: color,
        emissive: color,
        emissiveIntensity: intensity,
      });

      // Erstelle die Geometrie und das Mesh wie zuvor
      const geometry = new THREE.SphereGeometry(size, 32, 32);
      const sphere = new THREE.Mesh(geometry, material);
      sphere.position.set(data.x, data.y, data.z);
      scene.add(sphere);
      data.mesh = sphere;
    });

    camera.position.z = 1000;
    // camera.position.x = 5000;

    const controls = new OrbitControls(camera, renderer.domElement);

    // Render-Schleife
    const animate = function () {
      updateStarVisibility();
      requestAnimationFrame(animate);
      renderer.render(scene, camera);
    };
    animate();
    // ladeHauptSternzeichenSterne();
  });

  function toggleSonne() {
    sonneSichtbar = !sonneSichtbar;
    sonneMesh.visible = sonneSichtbar;
  }

  function updateStarVisibility() {
    stars.forEach((star) => {
      const isVisible =
        star.mag < magFilter &&
        (!showProperStarsOnly ||
          (star.proper !== null && star.proper !== undefined));
      if (star.mesh) {
        star.mesh.visible = isVisible;
      }
    });
  }
  $: updateStarVisibility(), [magFilter, showProperStarsOnly];

  async function loadStars() {
    const response = await axios.get("http://127.0.0.1:3001/stars/Vir");
    console.log(response.data);
    return response.data.map((star) => ({
      ...star,
      x: star.x0,
      y: star.y0,
      z: star.z0,
    }));
  }

  function getColorByCI(ci) {
    if (ci < 0)
      return 0x9db4ff; // Blau
    else if (ci < 0.5)
      return 0xbcd2ff; // Hellblau
    else if (ci < 1.0)
      return 0xfbfbfb; // Weiß
    else if (ci < 1.5)
      return 0xfff4ea; // Gelblich
    else return 0xffd2a1; // Orange/Rot
  }

  function getSizeByAbsMag(absmag) {
    if (absmag < 0)
      return 2; // Sehr hell, also größer
    else if (absmag < 5)
      return 1.5; // Hell
    else return 1; // Standardgröße
  }

  function getIntensityByMag(mag) {
    // Beispiel einer einfachen linearen Skalierung:
    // Hellerer Stern (niedriger 'mag') hat höhere Intensität
    const minMag = 0; // Minimal erwarteter 'mag'-Wert
    const maxMag = 10; // Maximal sinnvoller 'mag'-Wert für diese Skala
    const minIntensity = 0.1; // Minimale Intensität
    const maxIntensity = 1; // Maximale Intensität

    // Skaliere 'mag' auf den Intensitätsbereich
    if (mag > maxMag) mag = maxMag; // Begrenze den 'mag'-Wert, um Überintensitäten zu vermeiden
    const intensity =
      maxIntensity -
      ((mag - minMag) / (maxMag - minMag)) * (maxIntensity - minIntensity);
    return intensity;
  }
</script>

<main>
  <h1>{ausgewaehlteKonstellation}</h1>

  <button
    on:click={() => {
      showProperStarsOnly = !showProperStarsOnly;
    }}
  >
    {showProperStarsOnly ? "HS ausblenden" : "HS anzeigen"}</button
  >
  <!-- Slider im Markup -->
  <p>Mag-Filter</p>
  <div id="slider">
    <input type="range" min="0" max="20" bind:value={magFilter} />
    <p>{magFilter}</p>
  </div>
  <button on:click={toggleSonne}>
    {sonneSichtbar ? "Sonne ausblenden" : "Sonne anzeigen"}
  </button>

  <!-- Dropdown für Hauptsternzeichen -->
  <select bind:value={ausgewaehlteKonstellation}>
    <option value="">Wähle ein Hauptsternzeichen</option>
    {#each hauptKonstellationen as konstellation}
      <option value={konstellation.con}>{konstellation.con}</option>
    {/each}
  </select>

  <!-- Dropdown für andere Konstellationen -->
  <select bind:value={ausgewaehlteKonstellation}>
    <option value="">Wähle eine andere Konstellation</option>
    {#each andereKonstellationen as konstellation}
      <option value={konstellation.con}>{konstellation.con}</option>
    {/each}
  </select>

  <!-- <label>
      <input type="checkbox" bind:checked={alleHauptkonstellationenAnzeigen} on:change={toggleHauptkonstellationen} />
      Alle Hauptkonstellationen anzeigen
    </label>
    <p>Hauptkonstellationen Sterne</p>
    {#if hauptSternDaten.length > 0}
      <p>
        Sterne gefunden
      </p>
    {:else}
      <p>Keine Sterne gefunden.</p>
    {/if}
   -->
</main>

<svelte:head>
  <style>
    body {
      margin: 0;
    }
    canvas {
      display: block;
    }
  </style>
</svelte:head>

<!-- 
<script>
  let alleHauptkonstellationenAnzeigen = false;
  $: {
  if (stars.length > 0) {
    stars.forEach(star => {
      if (star.mesh) { // Stelle sicher, dass das Mesh-Objekt existiert
        // Aktualisiere die Sichtbarkeit basierend auf der Checkbox
        star.mesh.visible = alleHauptkonstellationenAnzeigen ? hauptSternzeichen.includes(star.con) : true;
      }
    });
  }
} 

  let hauptSternDaten = [];

async function ladeHauptSternzeichenSterne() {
  try {
    const response = await axios.get('http://127.0.0.1:3001/hauptkonstellationen');
    hauptSternDaten = response.data;
  } catch (error) {
    console.error('Fehler beim Laden der Hauptkonstellationen-Sterne: ', error);
  }
}

</script> -->

<style>
  #slider {
    display: flex;
    gap: 15px;
    margin-top: -30px;
  }
  main {
    /* position: absolute; */
    /* max-width: 100px;  */
    text-align: left;
  }
</style>
