<script>
  import { onMount } from "svelte";
  import axios from "axios";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
  import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
  import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
  import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
  import gsap from "gsap";
  import { arrays } from "./data/tierkreis";
  import Switch from './components/Switch.svelte'
	

  let raycaster = new THREE.Raycaster();
  let mouse = new THREE.Vector2();
  let centerKoords = {
    "x":0,
    "y":0,
    "z":0,
  };
  let sliderValue = false;
  let selectedArray;
  let centerAvailable = false;
  let camera, scene, renderer, controls;
  let composer, bloomPass, BLOOM_LAYER;
  const maxMag = 7;
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
    mag: -26.7,
    dist: 0,
  };
  let selectedStar = {
    id: 1,
    x: 0.000005,
    dist: 0,
    mag: -26.7,
    y: 0,
    z: 0,
    absmag: 4.85,
    ci: 0.9,
  };
  let sunIgnored = false;

  let lineGroup = new THREE.Group();

  onMount(() => {
    init();
    loadStars();
    // glow100();
  });

  function init() {
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      300
    );
    camera.position.z = 0.0001;

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000000); // Setzen Sie explizit eine Hintergrundfarbe

    scene.rotation.z = THREE.MathUtils.degToRad(337.5);

    renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement); // Stellen Sie sicher, dass dies ausgeführt wird
    controls = new OrbitControls(camera, renderer.domElement);

    composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));

    bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
    composer.addPass(bloomPass);
    BLOOM_LAYER = 1;
    camera.layers.enable(BLOOM_LAYER);
  }

  function loadStars() {
    const url = "https://starsapi.johannes-biess.com/all-stars";
    const body = { maxmag: maxMag };

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
            ra: star.ra,
            dec: star.dec,
            proper: star.proper,
          }));
        addStars(starsData);
        animate();
      })
      .catch((error) => {
        console.error("Fehler beim Abrufen der Sterndaten:", error);
      });
  }

  function glow100() {
    axios
      .get("https://starsapi.johannes-biess.com/top111")
      .then((response) => {
        const starIds = response.data; 
        applyGlowEffect(starIds); 
        starIds.forEach((star) => {
        });
      })
      .catch((error) => {
        console.error("Fehler beim Abrufen der Sterndaten:", error);
      });
}


  function applyGlowEffect(starIds) {
    scene.traverse((object) => {
    // Überprüfe, ob das Objekt ein Stern mit passender ID ist
    if (object.isMesh && object.userData && starIds.includes(object.userData.starData.id)) {
      console.log("Glowing star", object.userData.starData.id);
      object.layers.enable(BLOOM_LAYER);
    }
  });

}


  async function addStars(stars) {
    if (stars.length === 0) {
      console.error("Keine gültigen Sterndaten verfügbar.");
      return;
    }

    // Materialien definieren
    // const greenMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
    // const blueMaterial = new THREE.MeshBasicMaterial({ color: 0x0000ff });
    // const redMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });

    // // Geometrie definieren
    // const geometry = new THREE.BoxGeometry(0.1, 0.1, 0.1);

    // // Quader erstellen
    // const greenCube = new THREE.Mesh(geometry, greenMaterial);
    // greenCube.position.set(1, 0, 0);
    // scene.add(greenCube);

    // const blueCube = new THREE.Mesh(geometry, blueMaterial);
    // blueCube.position.set(0, 1, 0);
    // scene.add(blueCube);

    // const redCube = new THREE.Mesh(geometry, redMaterial);
    // redCube.position.set(0, 0, 1);
    // scene.add(redCube);

    console.log(stars);
    stars.forEach((star) => {
      if (star.id === 1 && sunIgnored == false) {
        sunIgnored = true;
        console.log("ignored");
        return;
      }

      let starGeometry;

      const originalRadius = berechneSternRadius(star.ci, star.absmag);
      let scaledRadius = mapRadius(
        originalRadius,
        minRadius,
        maxRadius,
        minNewRadius,
        maxNewRadius
      );
      // console.log(scaledRadius);
      let color = getColorByCI(star.ci);
      let intensity = getIntensityByMag(star.mag);
      if (star.id === 1) {
        color = 0xff0000;
        intensity = 8;
      }
      if (isNaN(scaledRadius)) {
        console.log("+");
        scaledRadius = 0.05;
      }
      if (star.dist < 200) {
        starGeometry = new THREE.SphereGeometry(scaledRadius, 16, 16);
      } else if (star.dist < 400) {
        starGeometry = new THREE.SphereGeometry(scaledRadius, 8, 8);
      } else {
        starGeometry = new THREE.SphereGeometry(scaledRadius, 4, 4);
      }

      const starMaterial = new THREE.MeshStandardMaterial({
        color: color,
        emissive: color,
        emissiveIntensity: intensity,
      });

      const sphere = new THREE.Mesh(starGeometry, starMaterial);
      sphere.position.set(star.y, star.z, star.x);
      // console.log("adding user data");
      sphere.userData.starData = {
        id: star.id,
        x: star.x,
        y: star.y,
        z: star.z,
        absmag: star.absmag,
        ci: star.ci,
        mag: star.mag,
        dist: star.dist,
        proper: star.proper,
      }; // Daten anhängen
      scene.add(sphere);
    });
    glow100();
  }

  function animate() {
    requestAnimationFrame(animate);
    updateVisibility();
    renderer.render(scene, camera);
    composer.render();
    controls.update();
  }

  function updateVisibility() {
    const frustum = new THREE.Frustum();
    const cameraViewProjectionMatrix = new THREE.Matrix4();

    // Aktualisiere die Frustum-Grenzen basierend auf der aktuellen Kameraposition und -konfiguration
    cameraViewProjectionMatrix.multiplyMatrices(
      camera.projectionMatrix,
      camera.matrixWorldInverse
    );
    frustum.setFromProjectionMatrix(cameraViewProjectionMatrix);

    // Durchlaufe alle Objekte der Szene und aktualisiere ihre Sichtbarkeit
    scene.traverse(function (object) {
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
      console.log("click registered");
      hideInfo();
      let firstObject = intersects[0].object;
      if (firstObject.userData.starData) {
        console.log(firstObject.userData.starData);
        selectedStar = {
          ...firstObject.userData.starData,
          object: firstObject,
        };
        console.log(selectedStar);
      }
    }
  }
  window.addEventListener("click", onMouseClick);

  async function jumpToStar() {
    const angle = THREE.MathUtils.degToRad(337.5);
    const rotationMatrix = new THREE.Matrix4().makeRotationZ(angle);
    let originalPosition = new THREE.Vector3(
      selectedStar.y,
      selectedStar.z,
      selectedStar.x + 0.01
    );

    let newPosition = originalPosition.applyMatrix4(rotationMatrix);

    // OrbitControls neu ausrichten
    controls.target.copy(newPosition);
    controls.update();

    await gsap.to(camera.position, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 2,
    });
    customLookAt(0,0,0);

    if (selectedStar.object) {
      scene.remove(selectedStar.object);
      disposeMaterial(selectedStar.object);
    }
    if (lastRemovedStar != null) {
      console.log("adding Stars", lastRemovedStar);
      addStars([lastRemovedStar]);
    }
    if (selectedStar == lastRemovedStar) {
      console.log("nothing happens");
    } else {
      console.log("selectedStar", selectedStar);
      console.log("lastRemovedStar", lastRemovedStar);
      lastRemovedStar = null;
      lastRemovedStar = selectedStar;
    }
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

  async function returnToSun() {  
    hideInfo();
    console.log("button pressed");
    let newPosition = new THREE.Vector3(0.000005, 0, 0);
    console.log(newPosition);
    await gsap.to(camera.position, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z + 0.01,
      duration: 2,
    }); // Etwas zurücksetzen, um den Stern zu betrachten

    // OrbitControls neu ausrichten
    controls.target.copy(newPosition);
    controls.update();
    selectedStar = {
      id: 1,
      x: 0.000005,
      dist: 0,
      mag: -26.7,
      y: 0,
      z: 0,
      absmag: 4.85,
      ci: 0.9,
    };
    if (lastRemovedStar != null) addStars([lastRemovedStar]);
    lastRemovedStar = {
      id: 1,
      x: 0.000005,
      dist: 0,
      mag: -26.7,
      y: 0,
      z: 0,
      absmag: 4.85,
      ci: 0.9,
    };
    console.log(scene.children);
    const sce = scene.children.reverse();
    const sun = sce.find((child) => {
      console.log("-");
      console.log(child);
      if (child.userData.starData.id === 1) console.log("hit");
      return child.userData.starData && child.userData.starData.id === 1;
    });
    scene.remove(sun);
    disposeMaterial(sun);
    if(centerAvailable) customLookAt(centerKoords.y, centerKoords.z, centerKoords.x);
    sliderValue = false;  
  }


  function getColorByCI(B_V) {
    // Berechnung der Temperatur basierend auf dem B-V Farbindex
    const temperature = 4600 * (1 / (0.92 * B_V + 1.7) + 1 / (0.92 * B_V + 0.62));

    // Zuordnung der Temperatur zu einer visuellen Farbe
    let color;
    if (temperature > 33000) {
        color = 0x9db4ff; // Blau
    } else if (temperature > 10000) {
        color = 0xcad8ff; // Blauweiß
    } else if (temperature > 7500) {
        color = 0xf8f7ff; // Weiß
    } else if (temperature > 6000) {
        color = 0xfff4ea; // Gelbweiß
    } else if (temperature > 5200) {
        color = 0xffd2a1; // Gelb
    } else if (temperature > 3700) {
        color = 0xffcc6f; // Orange
    } else {
        color = 0xff6f61; // Rot
    }

    return color;
}
  // function getColorByCI(ci) {
  //   if (ci < 0)
  //     return 0x9db4ff; // Blau
  //   else if (ci < 0.5)
  //     return 0xbcd2ff; // Hellblau
  //   else if (ci < 1.0)
  //     return 0xfbfbfb; // Weiß
  //   else if (ci < 1.5)
  //     return 0xfff4ea; // Gelblich
    // else return 0xffd2a1; // Orange/Rot
  // }

  function getIntensityByMag(mag) {
    const minMag = 0; // Minimal erwarteter 'mag'-Wert
    const maxMag = 10; // Maximal sinnvoller 'mag'-Wert für diese Skala
    const minIntensity = 0.5; // Minimale Intensität
    const maxIntensity = 2; // Maximale Intensität

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






  async function jumpToStar2(star) {
    console.log(selectedStar);
    console.log(star);
    selectedStar = star;
    const angle = THREE.MathUtils.degToRad(337.5);
    const rotationMatrix = new THREE.Matrix4().makeRotationZ(angle);
    let originalPosition = new THREE.Vector3(
      selectedStar.y,
      selectedStar.z,
      selectedStar.x + 0.01
    );

    let newPosition = originalPosition.applyMatrix4(rotationMatrix);

    // OrbitControls neu ausrichten
    controls.target.copy(newPosition);
    controls.update();
    const sce = scene.children.reverse();
    const starToRemove = sce.find((child) => {
      if (child.type === "Mesh") {
        console.log("-");
        console.log(child);
        if (child.userData.starData.id === selectedStar.id) console.log("hit");
        return (
          child.userData.starData &&
          child.userData.starData.id === selectedStar.id
        );
      }
    });
    await gsap.to(camera.position, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 2,
    });

    // const a = new THREE.Vector3( 0, 0, 0 );
    customLookAt( 0, 0, 0 );

    if (selectedStar.object) {
      scene.remove(selectedStar.object);
      disposeMaterial(selectedStar.object);
    }
    if (lastRemovedStar != null) {
      console.log("adding Stars", lastRemovedStar);
      addStars([lastRemovedStar]);
    }
    if (selectedStar == lastRemovedStar) {
      console.log("nothing happens");
    } else {
      console.log("selectedStar", selectedStar);
      lastRemovedStar = selectedStar;
    }

    scene.remove(starToRemove);
    disposeMaterial(starToRemove);
  }


  // $: if (selectedArray) {
  //   updateLines(selectedArray);
  // }

  function updateLines(arrayName) {
    lineGroup.clear(); // Löscht nur die Linien in der Gruppe
    console.log(arrayName);
    let array = arrays[arrayName];
    console.log(array);
    console.log(removeDuplicates(array));
    selectedArray = removeDuplicates(array);
    centerKoords = array[0];
    centerAvailable = true;
    console.log("centerkoords",centerKoords);
    if (array) {
      for (let i = 1; i < array.length - 1; i++) {
        addLine(array[i], array[i + 1]);
      }
      customLookAt(centerKoords.y,centerKoords.z,centerKoords.x);
    } else {
      console.error("Unbekanntes Array:", arrayName);
    }
  }

  function addLine(start, end) {
    let geometry = new THREE.BufferGeometry();
    let vertices = new Float32Array([
      start.y,
      start.z,
      start.x,
      end.y,
      end.z,
      end.x,
    ]);
    geometry.setAttribute("position", new THREE.BufferAttribute(vertices, 3));
    let color = Math.floor(Math.random() * 0xffffff);
    let material = new THREE.LineBasicMaterial({ color: color });
    let line = new THREE.Line(geometry, material);
    lineGroup.add(line); // Füge die Linie zur Gruppe hinzu
    scene.add(lineGroup);
    // console.log(center);
  }

  function moveToConstellation(newTarget) {
    // Aktuelle Position der Kamera speichern

    // Setze das neue Ziel für die OrbitControls
    controls.target.copy(newTarget);

    // Setze die Kameraposition zurück, um sicherzustellen, dass die Kamera nicht bewegt wird
    
    // camera.position.set(0, 0, 0.00005);

    // Notwendig, um die Änderungen zu verarbeiten
    controls.update();
  }

  function removeDuplicates(array) {
    const seenIds = new Map();
    let filteredArray = array.filter((item) => {
      if (item.id && !seenIds.has(item.id)) {
        seenIds.set(item.id, true);
        return true;
      }
      return false;
    });
    // console.log(constellationInfo);
    return filteredArray;
  }

  function resetTest() {
    togglePov();
    lineGroup.clear();
    centerAvailable = false;
  }

  let translateX = "-100%"; // Zustand der X-Translation des Containers

  function toggleContainer() {
    translateX = translateX === "-100%" ? "-73%" : "-100%"; // Schaltet zwischen den Positionen um
  }

  let infoContent = ""; // Inhalt für das info-overlay
  let displayInfoOverlay = "none"; // Steuert die Anzeige des info-overlays
  let showInfoOverlay = false;

  function showInfo(element) {
    // selectedArray = element;
    sliderValue = false;
    updateLines(element);
    infoContent = element;
    showInfoOverlay = true;
    displayInfoOverlay = "block"; // Zeigt das info-overlay an
    console.log("show info");
  }

  function hideInfo() {
    displayInfoOverlay = "none"; // Verbirgt das info-overlay
    showInfoOverlay = false;
  }


  function customLookAt(x, y, z) {
    const CameraPosition = camera.position;
    const controlPosition = controls.target;
    console.log(CameraPosition);
    console.log('Kameraposition:', camera?.position.y);
    console.log('Ziel der OrbitControls:', controls?.target);

    let dirVector = {
        x: x - controlPosition.x,
        y: y - controlPosition.y,
        z: z - controlPosition.z
    };

    let length = Math.sqrt(dirVector.x ** 2 + dirVector.y ** 2 + dirVector.z ** 2);

    dirVector.x /= length;
    dirVector.y /= length;
    dirVector.z /= length;

    const distance = 0.01;
    // let newPoint1 = {
    //     x: controlPosition.x + dirVector.x * distance,
    //     y: controlPosition.y + dirVector.y * distance,
    //     z: controlPosition.z + dirVector.z * distance
    // };
    let newPoint2 = {
        x: controlPosition.x - dirVector.x * distance,
        y: controlPosition.y - dirVector.y * distance,
        z: controlPosition.z - dirVector.z * distance
    };
    // console.log(newPoint2);
    camera.position.set(newPoint2.x, newPoint2.y, newPoint2.z);
}



function changeToPov() {
    const cameraPosition = camera.position;
    const controlPosition = controls.target;
    console.log('Kameraposition:', camera?.position);
    console.log('Ziel der OrbitControls:', controls?.target);

    let dirVector = {
        x: controlPosition.x - cameraPosition.x,
        y: controlPosition.y - cameraPosition.y,
        z: controlPosition.z - cameraPosition.z 
    };

    let length = Math.sqrt(dirVector.x ** 2 + dirVector.y ** 2 + dirVector.z ** 2);

    dirVector.x /= length;
    dirVector.y /= length;
    dirVector.z /= length;

    const distance = 0.01;
    let newPoint1 = {
        x: cameraPosition.x + dirVector.x * distance,
        y: cameraPosition.y + dirVector.y * distance,
        z: cameraPosition.z + dirVector.z * distance
    };
    // let newPoint2 = {
    //     x: cameraPosition.x - dirVector.x * distance,
    //     y: cameraPosition.y - dirVector.y * distance,
    //     z: cameraPosition.z - dirVector.z * distance
    // };
    console.log(newPoint1);
    moveToConstellation(new THREE.Vector3(newPoint1.x, newPoint1.y, newPoint1.z));
}

function togglePov() {
  //false == Pov-pov && true == orbit
  sliderValue = !sliderValue;
  if(sliderValue === true) {
    console.log("using  Orbit");
    moveToConstellation(new THREE.Vector3(centerKoords.y, centerKoords.z, centerKoords.x));
  } else {
    console.log("using POV");
    changeToPov();
  }
}
let name;

function submit() {
  const sce = scene.children.reverse();
  console.log(name);
    const starResult = sce.find((child) => {
      if (child.type === "Mesh") {
        // console.log("-");
        // console.log(child);
        if (child.userData.starData.proper === name) console.log(`found star ${name}`);
        return (
          child.userData.starData &&
          child.userData.starData.proper === name
        );
      }
    });
    console.log(starResult);
    //hier auf Stern ausrichten
}



  import svgURL from "./assets/constel.svg";
  import sunURL from "./assets/sun.svg";
</script>

<main>
  <div
    id="info-overlay"
    class="overlay"
    style="display: {showInfoOverlay ? 'flex' : 'none'} !important;"
  >
    {infoContent}

    {#if selectedArray && selectedArray.length > 0}
      {#each selectedArray as star}
        <button on:click={() => jumpToStar2(star)}>
          {star.id}
        </button>
      {/each}
    {/if}
  </div>
  <div id="container" style="--translateX: {translateX};">
    <div id="selection-overlay" class="overlay2">
      <button class="constbtn" on:click={() => resetTest()}
        ><img src={svgURL} /></button
      >
      <button class="constbtn" on:click={() => showInfo("Wassermann")}
        ><img src={svgURL} /></button
      >
      <button class="constbtn" on:click={() => showInfo("Widder")}
        ><img src={svgURL} /></button
      >
    </div>
    <div id="options">
      <button class="constbtn" on:click={toggleContainer}
        ><img src={svgURL} /></button
      >
      <button class="constbtn" on:click={returnToSun}
        ><img src={sunURL} /></button
      >
      <button class="constbtn"><img src={sunURL} on:click={() => {customLookAt(0,0,0)}}/></button>
    </div>
  </div>
  <!-- <select bind:value={selectedArray}>
    <option value="">Tierkreiszeichen</option>
    {#each Object.keys(arrays) as arrayName}
      <option value={arrayName}>{arrayName}</option>
    {/each}
  </select>

  <button on:click={returnToSun}>Sonne</button> -->
  <div
    class="overlay"
    id="singleStar-overlay"
    style="display: {showInfoOverlay ? 'none' : 'block'} !important;"
  >
    <h2>Stern: {selectedStar.id}</h2>
    <p>Magnitude: {selectedStar.mag}</p>
    <p>Entfernung: {selectedStar.dist} Lichtjahre</p>
    <button on:click={jumpToStar}>Sprung</button>
  </div>
  <div id='search'>
    <input bind:value={name} placeholder="enter name of star" />
    <button on:click={submit}>Submit</button>
  </div>
  <button id="moin" style="background-color: {sliderValue ? 'red' : 'green'} !important; display: {centerAvailable ? 'block' : 'none'}" on:click={togglePov}>toggle</button>
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
  #search {
    position: absolute;
    top:50px;
    left:10px;

  }
  #moin {
    position: absolute;
    top: 500px;
  }
  .constbtn {
    padding: 5px;
    background-color: transparent;
  }
  img {
    width: 25px;
    height: 25px;
  }
  main {
    position: absolute;
    width: 100%;
    /* border: 2px solid pink;   */
    display: flex;
    /* height: 98vh; */
    align-items: end;
    /* justify-content: center; */
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
  #container {
    display: flex;
    position: absolute;
    z-index: 20;
    width: auto;
    top: 0px;
    left: 150px;
    transform: translateX(var(--translateX));
    transition: transform 0.5s ease;
  }

  #selection-overlay {
    display: flex;
    flex-direction: column;
    border-bottom-right-radius: 10px;
  }

  .overlay2 {
    display: flex;
    background-color: rgba(255, 255, 255, 0.1);
    padding-left: 10px;
    padding-right: 10px;
  }

  .glow {
    box-shadow: 0 0 20px rgb(3, 164, 0); /* Anpassen für den gewünschten Effekt */
  }

  #options {
    padding-left: 10px;
    padding-right: 10px;
    background-color: rgba(255, 255, 255, 0.1);
    color: white;
    max-height: 40px;
    border-bottom-right-radius: 10px;
    margin: 0;
    display: flex;
    gap: 10px;
  }

  #info-overlay {
    position: absolute;
    /* display: var(--displayInfoOverlay, none); */
    right: 10px;
    top: 10px;
    /* display: flex; */
    flex-direction: column;
    align-content: center;
  }
  
  
</style>
