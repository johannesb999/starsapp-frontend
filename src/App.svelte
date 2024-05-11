<script>
  import { onMount } from "svelte";
  import axios from "axios";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

  let raycaster = new THREE.Raycaster();
  let mouse = new THREE.Vector2();

  let container;
  let camera, scene, renderer, controls;
  let frustum;
  let starField;
  const maxDiameter = 0.3;
  let selectedConstellation = "";
  let selectedConstellation2 = "";
  const maxMag = 8;
  // Beispielwerte, die Sie erhalten haben
  const minRadius = 0.17;
  const maxRadius = 1587.37;
  const minNewRadius = 0.05; // Mindestgröße für Sichtbarkeit
  const maxNewRadius = 0.3; // Maximalgröße für die Darstellung
  let lastRemovedStar = {
    id: 1,
    x: 0.000005,
    y: 0,
    z: 0,
    absmag: 4.85,
    ci: 0.9,
  };
  let selectedStar = { id: 1, x: 0.000005, y: 0, z: 0, absmag: 4.85, ci: 0.9 };
  let sunIgnored = false;

  let lastCameraPosition = new THREE.Vector3();
  let lastCameraQuaternion = new THREE.Quaternion();


  const constellations = [
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
  const constellationsRank2 = [
    "And",
    "Ant",
    "And",
    "Ant",
    "Aps",
    "Aql",
    "Aqr",
    "Ara",
    "Ari",
    "Aur",
    "Boo",
    "CMa",
    "CMi",
    "CVn",
    "Cae",
    "Cam",
    "Cap",
    "Car",
    "Cas",
    "Cen",
    "Cep",
    "Cet",
    "Cha",
    "Cir",
    "Cnc",
    "Col",
    "Com",
    "CrA",
    "CrB",
    "Crt",
    "Cru",
    "Crv",
    "Cyg",
    "Del",
    "Dor",
    "Dra",
    "Eco",
    "Equ",
    "Eri",
    "For",
    "Gem",
    "Gru",
    "Her",
    "Hor",
    "Hya",
    "Hyi",
    "Ind",
    "LMi",
    "Lac",
    "Leo",
    "Lep",
    "Lib",
    "Lup",
    "Lyn",
    "Lyr",
    "Men",
    "Mic",
    "Mon",
    "Mus",
    "Nor",
    "Oct",
    "Oph",
    "Ori",
    "Pav",
    "Peg",
    "Per",
    "Phe",
    "Pic",
    "PsA",
    "Psc",
    "Pup",
    "Pyx",
    "Ret",
    "Scl",
    "Sco",
    "Sct",
    "Ser",
    "Sex",
    "Sge",
    "Sgr",
    "Sir",
    "Tau",
    "Tel",
    "TrA",
    "Tri",
    "Tuc",
    "UMa",
    "UMi",
    "Vel",
    "Vir",
    "Vol",
    "Vul",
  ];

  onMount(() => {
    init();
    loadStars();
  });

  function init() {
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      500
    );
    camera.position.z = 0.0001;
    // camera.position.set(0, 0, 0)

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000000); // Setzen Sie explizit eine Hintergrundfarbe

    renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement); // Stellen Sie sicher, dass dies ausgeführt wird
    controls = new OrbitControls(camera, renderer.domElement);
  }

  function loadStars(selected) {
    const url = selectedConstellation
      ? "https://starsapi.johannes-biess.com/stars"
      : "https://starsapi.johannes-biess.com/all-stars";
    const body = { maxmag: maxMag, constellation: selected };

    axios
      .post(url, body)
      .then((response) => {
        scene.clear(); // Leert die Szene vor dem Hinzufügen neuer Sterne
        console.log(response.data);
        const starsData = response.data
          .filter(
            (star) =>
              star.x0 !== undefined &&
              star.y0 !== undefined &&
              star.z0 !== undefined
          )
          .map((star) => ({
            x: star.x0,
            y: star.y0,
            z: star.z0,
            id: star.id,
            absmag: star.absmag,
            ci: star.ci,
            mag: star.mag,
            dist: star.dist,
          }));
        addStars(starsData);
        animate();
      })
      .catch((error) => {
        console.error("Fehler beim Abrufen der Sterndaten:", error);
      });
  }

  function handleConstellationChange(event) {
    selectedConstellation = event.target.value;
    removeStars();
    renderer.renderLists.dispose();
    scene.clear();
    loadStars(selectedConstellation);
  }
  function handleConstellationChange2(event) {
    selectedConstellation2 = event.target.value;
    removeStars();
    renderer.renderLists.dispose();
    scene.clear();
    loadStars(selectedConstellation2);
  }

  function removeStars() {
    scene.children.forEach((child) => {
      if (child.userData.starData) {
        // Überprüfen Sie, ob es sich um einen Stern handelt
        scene.remove(child);
        if (child.geometry) child.geometry.dispose();

        if (child.material) {
          if (Array.isArray(child.material)) {
            child.material.forEach((material) => {
              if (material.map) material.map.dispose();
              material.dispose();
            });
          } else {
            if (child.material.map) child.material.map.dispose();
            child.material.dispose();
          }
        }
      }
    });
  }

  function addStars(stars) {
    if (stars.length === 0) {
      console.error("Keine gültigen Sterndaten verfügbar.");
      return;
    }

    console.log(stars);
    stars.forEach((star) => {
      if (star.id === 1 && sunIgnored == false) {
        sunIgnored = true;
        return;
      }

      const originalRadius = berechneSternRadius(star.ci, star.absmag);
      let scaledRadius = mapRadius(
        originalRadius,
        minRadius,
        maxRadius,
        minNewRadius,
        maxNewRadius  
      );
      let color = getColorByCI(star.ci);
      let intensity = getIntensityByMag(star.mag);
      if (star.id === 1) {
        color = 0xff0000;
        intensity = 0xff0000;
      }
      if (isNaN(scaledRadius)) {
        console.log("+");
        scaledRadius = 0.05;
      }
      const starGeometry = new THREE.SphereGeometry(scaledRadius, 16, 16);

      const starMaterial = new THREE.MeshStandardMaterial({
        color: color,
        emissive: color,
        emissiveIntensity: intensity,
      });

      const sphere = new THREE.Mesh(starGeometry, starMaterial);
      sphere.position.set(star.x, star.y, star.z);
      sphere.userData.starData = {
        id: star.id,
        x: star.x,
        y: star.y,
        z: star.z,
        absmag: star.absmag,
        ci: star.ci,
        mag: star.mag,
        dist: star.dist,
      }; // Daten anhängen
      scene.add(sphere);
    });
  }

  function animate() {
    requestAnimationFrame(animate);
    updateVisibility();
    renderer.render(scene, camera); 
    controls.update();
  }

  function cameraMoved() {
    let positionChanged = !lastCameraPosition.equals(camera.position);
    let quaternionChanged = !lastCameraQuaternion.equals(camera.quaternion);

    // Aktualisiere die letzte bekannte Position und Ausrichtung der Kamera für den nächsten Frame
    lastCameraPosition.copy(camera.position);
    lastCameraQuaternion.copy(camera.quaternion);

    return positionChanged || quaternionChanged;
}

  function updateVisibility() {
  const frustum = new THREE.Frustum();
  const cameraViewProjectionMatrix = new THREE.Matrix4();

  // Aktualisiere die Frustum-Grenzen basierend auf der aktuellen Kameraposition und -konfiguration
  cameraViewProjectionMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
  frustum.setFromProjectionMatrix(cameraViewProjectionMatrix);

  // Durchlaufe alle Objekte der Szene und aktualisiere ihre Sichtbarkeit
  scene.traverse(function(object) {
    if (object instanceof THREE.Mesh) {
      object.visible = frustum.intersectsObject(object);
    }
  });
}


  function onMouseClick(event) {
    // Berechnen der Mausposition im Normalized Device Coordinate (NDC) Raum
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    // Aktualisieren des Raycasters mit der Kamera und Mausposition
    raycaster.setFromCamera(mouse, camera);

    // Berechnen von Objekten, die vom Raycaster geschnitten werden
    const intersects = raycaster.intersectObjects(scene.children);

    if (intersects.length > 0) {
      let firstObject = intersects[0].object;
      if (firstObject.userData.starData) {
        console.log(firstObject.userData.starData);

        if (lastRemovedStar != null) addStars([lastRemovedStar]);

        lastRemovedStar = null;

        selectedStar = {
          ...firstObject.userData.starData,
          object: firstObject,
        };
        // let newPosition = new THREE.Vector3(firstObject.userData.starData.x, firstObject.userData.starData.y, firstObject.userData.starData.z);
        // camera.position.copy(newPosition);
        // camera.position.setZ(newPosition.z + 0.2); // Etwas zurücksetzen, um den Stern zu betrachten

        // // OrbitControls neu ausrichten
        // controls.target.copy(newPosition);
        // controls.update();
      }
    }
  }
  window.addEventListener("click", onMouseClick);
  function jumpToStar() {
    let newPosition = new THREE.Vector3(
      selectedStar.x,
      selectedStar.y,
      selectedStar.z
    );
    camera.position.copy(newPosition);
    camera.position.setZ(newPosition.z + 0.001);
    // OrbitControls neu ausrichten
    controls.target.copy(newPosition);
    controls.update();

    if (selectedStar.object) {
      scene.remove(selectedStar.object);
      disposeMaterial(selectedStar.object);
    }

    // Aktualisieren von lastRemovedStar
    lastRemovedStar = selectedStar;
  }

  function returnToSun() {
    console.log("button pressed");
    let newPosition = new THREE.Vector3(0.000005, 0, 0);
    console.log(newPosition);
    camera.position.copy(newPosition);
    camera.position.setZ(newPosition.z + 0.1); // Etwas zurücksetzen, um den Stern zu betrachten

    // OrbitControls neu ausrichten
    controls.target.copy(newPosition);
    controls.update();
    selectedStar = { id: 1, x: 0.000005, y: 0, z: 0, absmag: 4.85, ci: 0.9 };
    if (lastRemovedStar != null) addStars([lastRemovedStar]);
    lastRemovedStar = { id: 1, x: 0.000005, y: 0, z: 0, absmag: 4.85, ci: 0.9 };
    console.log(scene.children);
    const sce = scene.children.reverse();
    const sun = sce.find((child) => {
      console.log("-");
      if (child.userData.starData.id === 1) console.log("hit");
      return child.userData.starData && child.userData.starData.id === 1;
    });
    scene.remove(sun);
    disposeMaterial(sun);
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

  function berechneSternRadius(CI, M) {
    // if(CI == undefined || M == undefined) console.log("CI" + CI + "M" +M);
    const L_sonne = 3.828e26; // Leuchtkraft der Sonne in Watt
    const sigma = 5.67e-8; // Stefan-Boltzmann-Konstante in W/m^2/K^4
    const pi = Math.PI;

    const T = 4600 * (1 / (0.92 * CI + 1.7) + 1 / (0.92 * CI + 0.62));
    const L = L_sonne * Math.pow(10, 0.4 * (4.83 - M));
    const R = Math.sqrt(L / (4 * pi * sigma * Math.pow(T, 4)));
    const R_sonnen = R / 6.96e8; // Umrechnung in Sonnenradien

    return R_sonnen;
  }

  function mapRadius(originalRadius, minOriginal, maxOriginal, minNew, maxNew) {
    // Skalieren des Radius innerhalb des neuen Bereichs
    return (
      ((originalRadius - minOriginal) / (maxOriginal - minOriginal)) *
        (maxNew - minNew) +
      minNew
    );
  }

  function disposeMaterial(object) {
    if (object.geometry) object.geometry.dispose();
    if (object.material) {
      if (Array.isArray(object.material)) {
        object.material.forEach((material) => material.dispose());
      } else {
        object.material.dispose();
      }
    }
  }
</script>

<main>
  <select on:change={handleConstellationChange}>
    <option value="">Alle Sterne anzeigen</option>
    {#each constellations as constellation}
      <option value={constellation}>{constellation}</option>
    {/each}
  </select>
  <select on:change={handleConstellationChange2}>
    <option value="">Alle Sterne anzeigen</option>
    {#each constellationsRank2 as constellation}
      <option value={constellation}>{constellation}</option>
    {/each}
  </select>
  <button on:click={returnToSun}>zur Sonne zurück</button>
  <div class="overlay">
    <h2>Stern: {selectedStar.id}</h2>
    <p>Magnitude: {selectedStar.mag}</p>
    <p>Entfernung: {selectedStar.dist} Lichtjahre</p>
    <button on:click={jumpToStar}>Sprung</button>
  </div>
</main>

<svelte:head>
  <style>
    body {
      margin: 0;
      min-width: 0px;
      width: 0px !important;
    }
    canvas {
      display: block;
    }
  </style>
</svelte:head>

<style>
  main {
    position: absolute;
    width: 100%;
    /* border: 2px solid pink;   */
    display: flex;
    justify-content: start;
    gap: 5px;
  }
  .overlay {
    position: absolute;
    top: 10px;
    right: 10px;
    background-color: rgba(255, 255, 255, 0.1);
    padding: 10px;
    border: 2px solid black;
    border-radius: 8px;
    cursor: pointer;
  }
</style>
