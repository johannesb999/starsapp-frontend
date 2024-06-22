<script>
  import { onMount } from "svelte";
  import axios from "axios";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
  import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
  import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
  import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";
  // @ts-ignore einfach löschen für fehleranzeige
  import gsap from "gsap";
  import { arrays } from "./data/tierkreis";
  import Switch from "./components/Switch.svelte";

  let raycaster = new THREE.Raycaster();
  let mouse = new THREE.Vector2();
  let centerKoords = {
    x: 0,
    y: 0,
    z: 0,
  };
  let toggleValue = false;
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
    proper: "Sun",
    hip: 1,
    wikiUrl: "https://de.wikipedia.org/wiki/Sonne",
  };
  let selectedStar = {
    id: 1,
    x: 0.000005,
    y: 0,
    z: 0,
    absmag: 4.85,
    ci: 0.9,
    mag: -26.7,
    dist: 0,
    proper: "Sun",
    hip: 1,
    wikiUrl: "https://de.wikipedia.org/wiki/Sonne",
  };
  let sunIgnored = false;

  let lineGroup = new THREE.Group();

  onMount(() => {
    init();
    loadStars();
  });

  function init() {
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      300
    ); // @ts-ignore einfach löschen für fehleranzeige
    camera.position.z = 0.0001;

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000000); // Setzen Sie explizit eine Hintergrundfarbe
    // @ts-ignore einfach löschen für fehleranzeige
    scene.rotation.z = THREE.MathUtils.degToRad(337.5);

    renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement); // Stellen Sie sicher, dass dies ausgeführt wird
    controls = new OrbitControls(camera, renderer.domElement);

    composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));

    bloomPass = new UnrealBloomPass(
      new THREE.Vector2(window.innerWidth, window.innerHeight),
      1.5,
      0.4,
      0.85
    );
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
        // console.log(response.data);
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
            wikiUrl: star.wikiUrl,
            hip: star.hip,
          }));
        addStars(starsData);
        animate();
      })
      .catch((error) => {
        console.error("Fehler beim Abrufen der Sterndaten:", error);
      });
  }

  async function addStars(stars) {
    if (stars.length === 0) {
      console.error("Keine gültigen Sterndaten verfügbar.");
      return;
    }

    // console.log(stars);
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
      // @ts-ignore einfach löschen für fehleranzeige
      const sphere = new THREE.Mesh(starGeometry, starMaterial);
      // @ts-ignore einfach löschen für fehleranzeige
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
        hip: star.hip,
        wikiUrl: star.wikiUrl,
      }; // Daten anhängen
      scene.add(sphere);
    });
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
      let firstObject = intersects[0].object;
      if (firstObject.userData.starData) {
        console.log(firstObject.userData.starData);
        selectedStar = {
          ...firstObject.userData.starData,
          object: firstObject,
        };
        console.log(selectedStar);
        showInfoBox = true; // Info-Box anzeigen, wenn ein Stern angeklickt wird
      }
    } else {
      // Wenn kein Stern getroffen wird, die Info-Box ausblenden
      showInfoBox = false;
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
    customLookAt(0, 0, 0);

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
      proper: "Sun",
      hip: 1,
      wikiUrl: "https://de.wikipedia.org/wiki/Sonne",
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
      proper: "Sun",
      hip: 1,
      wikiUrl: "https://de.wikipedia.org/wiki/Sonne",
    };

    console.log("lastRemovedStar", lastRemovedStar);
    // console.log(scene.children);
    const sce = scene.children.reverse();
    const sun = sce.find((child) => {
      console.log("-");
      // console.log(child);
      if (child.userData.starData.id === 1) console.log("hit");
      return child.userData.starData && child.userData.starData.id === 1;
    });
    scene.remove(sun);
    disposeMaterial(sun);
    if (centerAvailable)
      customLookAt(centerKoords.y, centerKoords.z, centerKoords.x);
    toggleValue = false;
  }

  function getColorByCI(B_V) {
    // Berechnung der Temperatur basierend auf dem B-V Farbindex
    const temperature =
      4600 * (1 / (0.92 * B_V + 1.7) + 1 / (0.92 * B_V + 0.62));

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
        // console.log(child);
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
    customLookAt(0, 0, 0);

    if (selectedStar.object) {
      scene.remove(selectedStar.object);
      disposeMaterial(selectedStar.object);
    }
    if (lastRemovedStar != null) {
      // console.log("adding Stars", lastRemovedStar);
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
    // console.log(array);
    // console.log(removeDuplicates(array));
    selectedArray = removeDuplicates(array);
    centerKoords = array[0];
    centerAvailable = true;
    // console.log("centerkoords", centerKoords);
    if (array) {
      for (let i = 1; i < array.length - 1; i++) {
        addLine(array[i], array[i + 1]);
      }
      customLookAt(centerKoords.y, centerKoords.z, centerKoords.x);
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
    let color = 0xffff; //Math.floor(Math.random() * 0xffffff);
    let material = new THREE.LineBasicMaterial({ color: color });
    let line = new THREE.Line(geometry, material);
    lineGroup.add(line); // Füge die Linie zur Gruppe hinzu
    scene.add(lineGroup);
    // console.log(center);
  }

  // Aktuelle Position der Kamera speichern
  function moveToConstellation(newTarget) {
    controls.target.copy(newTarget); // Setze das neue Ziel für die OrbitControls

    // Setze die Kameraposition zurück, um sicherzustellen, dass die Kamera nicht bewegt wird
    // camera.position.set(0, 0, 0.00005);
    controls.update(); // Notwendig, um die Änderungen zu verarbeiten
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

  let showZodiacSelection = false;

  function toggleZodiacSelectionContainer() {
    showZodiacSelection = !showZodiacSelection;
    // Automatisch die zodiacInfosBox ausblenden, wenn der Container geschlossen wird
    if (!showZodiacSelection) {
      showzodiacInfosBox = false;
      resetTest();
    }
  }

  let infoContent = ""; // Inhalt für das zodiacInfosBox
  // let displaySmallBox = "none"; // Steuert die Anzeige des zodiacInfosBoxs
  let showzodiacInfosBox = false;
  let showInfoBox = false; //steuert die Anzeige der lookAtStarInfoBox

  function showInfo(element) {
    // selectedArray = element;
    toggleValue = false;
    updateLines(element);
    infoContent = element;
    showzodiacInfosBox = true;
    // displaySmallBox = "block"; // Zeigt das zodiacInfosBox an
    if (!showZodiacSelection) {
      showZodiacSelection = true;
    }
    console.log("show info");
  }

  function hideInfo() {
    // displaySmallBox = "none"; // Verbirgt das zodiacInfosBox
    showzodiacInfosBox = false;
    console.log("hide info");
  }

  function customLookAt(x, y, z) {
    const CameraPosition = camera.position;
    const controlPosition = controls.target;
    console.log(CameraPosition);
    console.log("Kameraposition:", camera?.position.y);
    console.log("Ziel der OrbitControls:", controls?.target);

    let dirVector = {
      x: x - controlPosition.x,
      y: y - controlPosition.y,
      z: z - controlPosition.z,
    };

    let length = Math.sqrt(
      dirVector.x ** 2 + dirVector.y ** 2 + dirVector.z ** 2
    );

    dirVector.x /= length;
    dirVector.y /= length;
    dirVector.z /= length;

    const distance = 0.01;

    let newPoint2 = {
      x: controlPosition.x - dirVector.x * distance,
      y: controlPosition.y - dirVector.y * distance,
      z: controlPosition.z - dirVector.z * distance,
    };
    // console.log(newPoint2);
    camera.position.set(newPoint2.x, newPoint2.y, newPoint2.z);
  }

  function changeToPov() {
    const cameraPosition = camera.position;
    const controlPosition = controls.target;
    console.log("Kameraposition:", camera?.position);
    console.log("Ziel der OrbitControls:", controls?.target);

    let dirVector = {
      x: controlPosition.x - cameraPosition.x,
      y: controlPosition.y - cameraPosition.y,
      z: controlPosition.z - cameraPosition.z,
    };

    let length = Math.sqrt(
      dirVector.x ** 2 + dirVector.y ** 2 + dirVector.z ** 2
    );

    dirVector.x /= length;
    dirVector.y /= length;
    dirVector.z /= length;

    const distance = 0.01;
    let newPoint1 = {
      x: cameraPosition.x + dirVector.x * distance,
      y: cameraPosition.y + dirVector.y * distance,
      z: cameraPosition.z + dirVector.z * distance,
    };

    console.log(newPoint1);
    moveToConstellation(
      new THREE.Vector3(newPoint1.x, newPoint1.y, newPoint1.z)
    );
  }

  function togglePov() {
    //false == Pov-pov && true == orbit
    toggleValue = !toggleValue;
    if (toggleValue === true) {
      console.log("using  Orbit");
      moveToConstellation(
        new THREE.Vector3(centerKoords.y, centerKoords.z, centerKoords.x)
      );
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
        if (child.userData.starData.proper === name)
          console.log(`found star ${name}`);
        return (
          child.userData.starData && child.userData.starData.proper === name
        );
      }
    });
    // console.log(starResult);
    //hier auf Stern ausrichten
  }

  import constellation from "./assets/constel.svg";
  import SunIcon from "./assets/sun.svg";
  import Fische from "./assets/Fische.svg";
  import Jungfrau from "./assets/Jungfrau.svg";
  import Krebs from "./assets/Krebs.svg";
  import Löwe from "./assets/Löwe.svg";
  import Schütze from "./assets/Schütze.svg";
  import Skorpion from "./assets/Skorpion.svg";
  import Stier from "./assets/Stier.svg";
  import Waage from "./assets/Waage.svg";
  import Wassermann from "./assets/Wassermann.svg";
  import Widder from "./assets/Widder.svg";
  import Zwillinge from "./assets/Zwillinge.svg";
  import Steinbock from "./assets/Steinbock.svg";
  import Teleskop from "./assets/Teleskop.svg";
  import Lupe from "./assets/Lupe.svg";
  import Erde from "./assets/Erde.svg";

  let headerIndex = 0;
  const headerMappings = [
    { key: "id", label: "ID" },
    { key: "proper", label: "Proper" },
    { key: "hip", label: "HIP" },
  ];

  // function toggleinfoBoxHeader() {
  //   headerIndex = (headerIndex + 1) % headerMappings.length;
  // }

  //infoBoxLookAtStarTitle
  $: currentHeader = selectedStar.proper
    ? `Name: ${selectedStar.proper}`
    : `HIP-Katalog: ${selectedStar.hip || "Nicht Verfügbar"}`;
  // lastRemovedStar
  $: currentHeaderLastRemoved = lastRemovedStar.proper
    ? `Aktuell auf ${lastRemovedStar.proper}`
    : `HIP-Katalog: ${lastRemovedStar.hip || "Nicht Verfügbar"}`;

  function toggleinfoBoxHeaderLastRemoved() {
    headerIndex = (headerIndex + 1) % headerMappings.length;
  }

  let showSearch = false;
  let searchRef;

  function toggleSearch(event) {
    showSearch = !showSearch;
    event.stopPropagation(); // Verhindert das Auslösen von handleOutsideClick beim ersten Klick

    if (showSearch) {
      // Fügt einen Event-Listener hinzu, wenn die Suchleiste gezeigt wird
      setTimeout(() => window.addEventListener("click", handleOutsideClick), 0);
    }
  }

  function handleOutsideClick(event) {
    if (searchRef && !searchRef.contains(event.target)) {
      showSearch = false;
      window.removeEventListener("click", handleOutsideClick);
    }
  }
</script>

<!-- HTML -->
<main>
  <div
    id="zodiacInfosBox"
    class="overlay"
    style="display: {showzodiacInfosBox ? 'flex' : 'none'} !important;"
  >
    <!-- Stellen Sie sicher, dass alle Inhalte hier innerhalb sind -->
    <div class="zodiacStarLinksHeader">{infoContent}</div>
    {#if selectedArray && selectedArray.length > 0}
      {#each selectedArray as star}
        <button class="jumpToStar2Button" on:click={() => jumpToStar2(star)}>
          {star.id}
        </button>
      {/each}
    {/if}
  </div>

  <div
    id="zodiacSelectionContainer"
    style="--translateX: {showZodiacSelection ? '0%' : '-100%'};"
  >
    <div id="zodiacSelectionOverlay">
      <button class="zodiacButtons" on:click={() => showInfo("Steinbock")}
        ><img class="svgIcon" src={Steinbock} alt="Steinbock" />
        <span>Steinbock</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Wassermann")}
        ><img class="svgIcon" src={Wassermann} alt="Wassermann" />
        <span>Wassermann</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Fische")}
        ><img class="svgIcon" src={Fische} alt="Fische" />
        <span>Fische</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Widder")}
        ><img class="svgIcon" src={Widder} alt="Widder" />
        <span>Widder</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Stier")}
        ><img class="svgIcon" src={Stier} alt="Stier" /> <span>Stier</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Zwillinge")}
        ><img class="svgIcon" src={Zwillinge} alt="Zwillinge" />
        <span>Zwillinge</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Krebs")}
        ><img class="svgIcon" src={Krebs} alt="Krebs" /> <span>Krebs</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Löwe")}
        ><img class="svgIcon" src={Löwe} alt="Löwe" /> <span>Löwe</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Jungfrau")}
        ><img class="svgIcon" src={Jungfrau} alt="Jungfrau" />
        <span>Jungfrau</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Waage")}
        ><img class="svgIcon" src={Waage} alt="Waage" /> <span>Waage</span>
      </button>
      <button class="zodiacButtons" on:click={() => showInfo("Skorpion")}
        ><img class="svgIcon" src={Skorpion} alt="Skorpion" />
        <span>Skorpion</span>
      </button>
      <!-- schlangenträger -->
      <button class="zodiacButtons" on:click={() => showInfo("Schütze")}
        ><img class="svgIcon" src={Schütze} alt="Schütze" />
        <span>Schhütze</span>
      </button>
      <!-- more -->
      <!-- ansicht in raster -->
    </div>
    <!-- <button class="quickSelectButtons" on:click={() => resetTest()}
      ><img class="svgIcon" src={constellation} alt="reset to sun" /></button
    > -->
    <!-- quickBar -->
    <div id="quickSelectBar">
      <button
        class="quickSelectButtons"
        on:click={toggleZodiacSelectionContainer}
      >
        <img
          class="quickSelectIcons"
          src={constellation}
          alt="Toggle visibility"
        />
      </button>
      <button class="quickSelectButtons" on:click={returnToSun}>
        <img class="quickSelectIcons" src={Erde} alt="Return to Sun" />
      </button>
      <button class="quickSelectButtons" on:click={() => customLookAt(0, 0, 0)}>
        <img class="quickSelectIcons" src={SunIcon} alt="Look at Sun" />
      </button>
      <button class="quickSelectButtons" on:click={toggleSearch}>
        <img class="quickSelectIcons" src={Lupe} alt="searchbar" />
      </button>
    </div>
  </div>
  <!-- infobox -->
  <div
    class="infoBoxLookAtStarTitle"
    style="display: {showInfoBox ? 'block' : 'none'}; !important;"
  >
    <div class="infoBoxHeader">
      {currentHeader}
    </div>
    <!-- weitere infos -->
    <p>Magnitude: {selectedStar.mag}</p>
    <p>Entfernung: {selectedStar.dist} Lichtjahre</p>
    <p>Farbindex: {selectedStar.ci}</p>
    <p>Leuchtkraft: {selectedStar.absmag}</p>

    <button id="jumpButton" on:click={jumpToStar}>Sprung</button>
    <button
      id="wikiLinkButton"
      on:click={() => window.open(selectedStar.wikiUrl, "_blank")}
      >Wikipedia</button
    >
  </div>

  <div
    class="infoBoxPosition"
    style="display: {showzodiacInfosBox ? 'none' : 'block'} !important;"
  >
    <div class="currentPosition">
      {currentHeaderLastRemoved}
    </div>
  </div>

  <div
    id="searchBar"
    bind:this={searchRef}
    style="display: {showSearch ? 'block' : 'none'};"
  >
    <input bind:value={name} placeholder="enter name of star" />
    <button on:click={submit}>Submit</button>
  </div>

  <button
    id="togglePovButton"
    style="display: {centerAvailable ? 'block' : 'none'}"
    on:click={togglePov}
  >
    {toggleValue ? "Orbit" : "POV"}
  </button>
</main>

<!-- CSS -->

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
  /* CSS */
  main {
    position: absolute;
    width: 100%;
    /* border: 2px solid pink;   */
    display: flex;
    /* height: 98vh; */
    align-items: end;
    /* justify-content: center; */
  }

  #togglePovButton {
    position: absolute;
    top: 3px;
    left: 50%;
    transform: showZodiacSelection (-50%);
    z-index: 100;
  }

  #quickSelectBar {
    padding-left: 50px;
    padding-right: 10px;
    background-color: rgba(166, 166, 166, 0.093);
    color: white;
    max-height: 70px;
    /* border-bottom-right-radius: 10px; */
    margin: 0;
    display: flex;
    /* gap: 10px; */
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    border-radius: 8px;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }

  .quickSelectButtons {
    padding: 20px;
    background-color: transparent;
    width: 100%;
    margin-bottom: 5px;
    border: none;
  }

  .quickSelectButtons:hover {
    background-color: rgba(
      255,
      255,
      255,
      0.2
    ); /* Hervorhebung beim Darüberfahren */
  }

  /* icons */
  .quickSelectIcons {
    width: 35px;
    height: 35px;
  }

  #searchBar {
    position: absolute;
    top: 75px;
    left: 300px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  #searchBar input {
    padding: 10px;
    border: 2px solid #b7b7b7;
    border-radius: 5px;
    margin-right: 10px;
  }

  #searchBar button {
    padding: 10px 20px;
    background-color: #8a8a8a;
    border: none;
    border-radius: 5px;
    color: #fff;
    cursor: pointer;
  }

  .zodiacButtons {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 10px 20px;
    background-color: transparent;
    width: 100%;
    border: none;
    color: white;
    text-align: center;
    cursor: pointer;
    transition: background-color 0.3s ease;
    position: relative;
  }

  .zodiacButtons:hover {
    background-color: rgba(
      255,
      255,
      255,
      0.2
    ); /* Hervorhebung beim Darüberfahren */
  }

  .svgIcon {
    width: 40px;
    height: 40px;
    border: none;
    transition: transform 0.3s ease;
    transform-origin: left center;
  }

  .svgIcon:hover {
    /* transform: scale(3) translateX(45px) translateY(5px); */
    border: none;
  }

  /* inhalt der Info box welche den Stern anzeigt den man anklickt */
  .infoBoxLookAtStarTitle {
    position: absolute;
    top: 10px;
    right: 10px;
    background-color: rgba(166, 166, 166, 0.093);
    padding: 10px;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    border-radius: 8px;
    /* cursor: pointer; */
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    text-align: left; /* Add this line to align the content to the left */
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }

  .infoBoxHeader {
    font-size: 24px; /* Größere Schrift */
    padding-bottom: 10px;
    font-weight: bold;

    color: white; /* Textfarbe */
    text-align: left;
  }
  #wikiLinkButton {
    background-color: transparent;
    border: none;
    color: white;
    padding: 15px 32px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    margin: 4px 2px;
    cursor: pointer;
  }
  #jumpButton {
    background-color: #cccccc;
    border: none;
    color: rgb(0, 0, 0);
    padding: 15px 32px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    margin: 4px 2px;
    cursor: pointer;
  }
  /* ende inhalt der Info box welche den Stern anzeigt den man anklickt ende */

  /* infobox die aktuelle Position enzeigt */
  .infoBoxPosition {
    position: fixed;
    bottom: 0px;
    left: 50%;
    transform: translate(-50%);
    background-color: rgba(166, 166, 166, 0.093);
    padding: 10px;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    border-radius: 8px;
    cursor: pointer;
    z-index: 100;
    min-height: 75px;
    min-width: 250px;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }

  .currentPosition {
    font-size: 28px;
    font-weight: bold;
    padding: 10px;
    color: white;
  }
  /* ende infobox die aktuelle Position enzeigt ende */

  /* sternzeichen auswahl Kontainer */
  #zodiacSelectionContainer {
    display: flex;
    position: absolute;
    z-index: 1000;
    width: 150px;
    top: 0px;
    left: 0px;
    transform: translateX(var(--translateX));
    transition: transform 0.5s ease;
  }

  #zodiacSelectionOverlay {
    display: flex;
    flex-direction: column;
    background-color: rgba(166, 166, 166, 0.093);
    padding: 10px;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    width: 100%;
    left: 0px;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }
  /* ende sternzeichen auswahl Kontainer ende */

  #zodiacInfosBox {
    position: absolute;
    top: 10px;
    right: 10px;
    background-color: rgba(166, 166, 166, 0.093);
    padding: 10px;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    border-radius: 8px;
    /* cursor: pointer; */
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    text-align: left; /* Add this line to align the content to the left */
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
    display: flex;
    flex-direction: column;
  }

  .zodiacStarLinksHeader {
    color: white;
    font-size: 28px;
  }

  .jumpToStar2Button {
    border-radius: 5px;
    cursor: pointer;
    gap: 5px;
    background-color: #cccccc;
    border: none;
    color: rgb(0, 0, 0);
    padding: 15px 32px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    margin: 4px 2px;
    cursor: pointer;
  }
</style>
