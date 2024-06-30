<script>
  import { onMount } from "svelte";
  import axios from "axios";
  import * as THREE from "three";
  import TWEEN from '@tweenjs/tween.js';
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
  import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
  import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
  import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";
  import gsap from "gsap";
  import { arrays } from "./data/tierkreis";
  import Switch from "./components/Switch.svelte";

  const darkZodiacMaterial = new THREE.LineBasicMaterial({ color:'#303030', transparent: true, opacity: 0})
  let raycaster = new THREE.Raycaster();
  let raycaster2 = new THREE.Raycaster();
  let mouse = new THREE.Vector2();
  let centerKoords = {
    x: 0,
    y: 0,
    z: 0,
  };
  let toggleValue = false;
  let drawn = false;
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
  let showCurrentInfosBox = true;
  let ZoddiacGroup = new THREE.Group();
  let BigZoddiacGroup = new THREE.Group();

  onMount(() => {
    init();
    loadStars();
    addCentralTorus();
  });

  function addCentralTorus() {}

  // Aufrufen der Funktion zum Hinzufügen des Torus

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
    renderer.setSize(window.innerWidth, window.innerHeight - 1);
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
            flam: star.flam,
            con: star.con,
            bayer: star.bayer,
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
      let color = getColorByCI(star.ci);
      let intensity = getIntensityByMag(star.absmag);
      if (star.id === 1) {
        color = '#FFC700';
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
      sphere.userData.starData = {
        x: star.x,
        y: star.y,
        z: star.z,
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
        flam: star.flam,
        con: star.con,
        bayer: star.bayer,
      }; 
      scene.add(sphere);
    });
    const radius = 290; 
    const points = 64; 

    const circlePoints = [];
    for (let i = 0; i <= points; i++) {
      const angle = (i / points) * 2 * Math.PI;
      circlePoints.push(
        new THREE.Vector3(Math.cos(angle) * radius, 0, Math.sin(angle) * radius)
      );
    }

    const lineGeometry = new THREE.BufferGeometry().setFromPoints(circlePoints);
    const lineMaterial = new THREE.LineBasicMaterial({ color: "#585801" });
    const line = new THREE.LineLoop(lineGeometry, lineMaterial);
    // line.layers.set(BLOOM_LAYER);

    line.rotation.y = -Math.PI / 2; // Drehe die Linie auf die XZ-Ebene
    scene.add(line);
    addAllLines();
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
    const minIntensity = 1; // Minimale Intensität
    const maxIntensity = 2.5; // Maximale Intensität

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

  function updateVisibility() {
    const frustum = new THREE.Frustum();
    const cameraViewProjectionMatrix = new THREE.Matrix4();

    cameraViewProjectionMatrix.multiplyMatrices(
      camera.projectionMatrix,
      camera.matrixWorldInverse
    );
    frustum.setFromProjectionMatrix(cameraViewProjectionMatrix);

    scene.traverse(function (object) {
      if (object instanceof THREE.Mesh) {
        object.visible = frustum.intersectsObject(object);
      }
    });
  }
  function animate() {
    TWEEN.update();
    requestAnimationFrame(animate);
    updateVisibility();
    renderer.render(scene, camera);
    composer.render();
    controls.update();
  }



  function onMouseClick(event) {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const meshObjects = [];
    scene.traverse(function (object) {
      if (object instanceof THREE.Mesh) {
        meshObjects.push(object);
      }
    });
    const intersects = raycaster.intersectObjects(meshObjects);
    if (intersects.length > 0) {
      let firstObject = intersects[0].object;
      if (firstObject.userData.starData) {
        selectedStar = {
          ...firstObject.userData.starData,
          object: firstObject,
        };
        console.log(selectedStar);
        showInfoBox = true; // Info-Box anzeigen, wenn ein Stern angeklickt wird
        showzodiacInfosBox = false;
      }
    } else {
      showInfoBox = false;
    }
  }
  window.addEventListener("click", onMouseClick);


function onMouseMove(event) {
  event.preventDefault();

  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  raycaster2.setFromCamera(mouse, camera);
  raycaster2.params.Points.threshold = 0.1; // Für PointCloud-Objekte, falls verwendet
  raycaster2.params.Mesh.threshold = 0.1; // Für Mesh-Objekte
  
  const intersects = raycaster2.intersectObjects(scene.children, true);
  const tooltip = document.getElementById('tooltip');
  if (intersects.length > 0) {
    if(intersects[0].object.type == "Mesh") {

      const firstObject = intersects[0].object;
      if (firstObject.userData.starData && firstObject.userData.starData.proper || firstObject.userData.starData.hip) {
        tooltip.style.display = 'block';
        tooltip.innerHTML = firstObject.userData.starData.proper 
  ? firstObject.userData.starData.proper 
  : firstObject.userData.starData.bayer 
    ? `${firstObject.userData.starData.bayer} ${firstObject.userData.starData.con}` 
    : firstObject.userData.starData.flam 
      ? `${firstObject.userData.starData.flam} ${firstObject.userData.starData.con}` 
      : `HIP: ${firstObject.userData.starData.hip}`;

        tooltip.style.left = event.clientX + 'px';
        tooltip.style.top = event.clientY - 40 + 'px';
      }
    }
  } else {
    tooltip.style.display = 'none';
  }
}

window.addEventListener('mousemove', onMouseMove);








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
    await gsap.to(controls.target, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 0.2,
      onUpdate: function () {
        controls.update();
      },
    });

    await gsap.to(camera.position, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 2,
    });
    showInfoBox = false;
    if (centerAvailable) {
      customLookAt(centerKoords.y, centerKoords.z, centerKoords.x);
    } else {
      customLookAt(0, 0, 0);
    }

    if (selectedStar != lastRemovedStar) {
      if (selectedStar.object) {
        scene.remove(selectedStar.object);
        disposeMaterial(selectedStar.object);
      }
      if (lastRemovedStar != null) {
        // console.log("adding Stars", lastRemovedStar);
        addStars([lastRemovedStar]);
      }
      if (selectedStar == lastRemovedStar) {
        // console.log("nothing happens");
      } else {
        // console.log("selectedStar", selectedStar);
        // console.log("lastRemovedStar", lastRemovedStar);
        lastRemovedStar = null;
        lastRemovedStar = selectedStar;
      }
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

    console.log(showZodiacSelection);
    if(showZodiacSelection) {
      fadeIn(darkZodiacMaterial);
      // fadeOut(materials[0]);
      // fadeOut(materials[1]);
      // fadeIn(materials[0]);
      // fadeIn(materials[1]);
    }
    let originalPosition;
    const angle = THREE.MathUtils.degToRad(337.5);
    const rotationMatrix = new THREE.Matrix4().makeRotationZ(angle);
    if(centerAvailable) {
      console.log(centerKoords);
      originalPosition = new THREE.Vector3(centerKoords.y, centerKoords.z, centerKoords.x);
    } else {
      originalPosition = new THREE.Vector3(0.000005, 0, 0);
    }

    let newPosition = originalPosition.applyMatrix4(rotationMatrix);
    console.log(newPosition)
    
    await gsap.to(controls.target, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 1,
      onUpdate: function () {
        controls.update();
      },
    });

    const cameraPosition = new THREE.Vector3(0.000005, 0, 0);

    await gsap.to(camera.position, {
      x: cameraPosition.x,
      y: cameraPosition.y,
      z: cameraPosition.z,
      duration: 2,
    });

    if(centerAvailable) {
      customLookAtControls(centerKoords.y, centerKoords.z, centerKoords.x);
    } else {
      customLookAt(0,0,0);
    }

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

    // console.log("lastRemovedStar", lastRemovedStar);
    // console.log(scene.children);
    const sce = scene.children.reverse();
    const sun = sce.find((child) => {
      if (child.type === "Mesh") {
        // console.log("-");
        // console.log(child);
        // if (child.userData.starData.id === 1) console.log("hit");
        return child.userData.starData && child.userData.starData.id === 1;
      }
    });
    scene.remove(sun);
    disposeMaterial(sun);
    // if (centerAvailable)
    //   customLookAt(centerKoords.y, centerKoords.z, centerKoords.x);
    // else customLookAt(0, 0, 0);
    toggleValue = false;
  }

 



  async function jumpToStar2(star) {
    fadeOut(darkZodiacMaterial);
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
    await gsap.to(controls.target, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 1,
      onUpdate: function () {
        controls.update();
      },
    });
    const sce = scene.children.reverse();
    const starToRemove = sce.find((child) => {
      if (child.type === "Mesh") {
        // console.log("-");
        // console.log(child);
        if (child.userData.starData.id === selectedStar.id) console.log("hit");
        return (
          child.userData.starData &&
          child.userData.starData.id === selectedStar.id
        );
      }
    });
    selectedStar = {
      ...selectedStar,
      object: starToRemove,
    };
    console.log(starToRemove);
    await gsap.to(camera.position, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 2,
    });

    // const a = new THREE.Vector3( 0, 0, 0 );
    customLookAt(0, 0, 0);

    if (selectedStar != lastRemovedStar) {
      if (starToRemove) {
        scene.remove(starToRemove);
        disposeMaterial(starToRemove);
      }
      if (lastRemovedStar != null) {
        // console.log("adding Stars", lastRemovedStar);
        addStars([lastRemovedStar]);
      }
      if (selectedStar == lastRemovedStar) {
        // console.log("nothing happens");
      } else {
        // console.log("selectedStar", selectedStar);
        lastRemovedStar = selectedStar;
        // console.log("lastRemoved", lastRemovedStar);
      }
    }

    // scene.remove(starToRemove);
    // disposeMaterial(starToRemove);
  }


  async function addAllLines() {
    console.log(typeof arrays);
    for (const key in arrays) {
      if (arrays.hasOwnProperty(key)) {
        // console.log(key, arrays[key]);
        const array = arrays[key];
        for (let i = 2; i < array.length - 1; i++) {
          await add2ndLine(array[i], array[i + 1]);
        }
      }
    }
  }

  async function add2ndLine(start, end) {
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
    let line = new THREE.Line(geometry, darkZodiacMaterial);
    ZoddiacGroup.add(line);
    scene.add(ZoddiacGroup);
  }

  let zodiacWiki = "";
  let currentMaterialIndex = 0;
  let materials = [];
  materials[0] = new THREE.LineBasicMaterial({ color: '#B8EEFF', transparent: true, opacity: 0 });
  materials[1] = new THREE.LineBasicMaterial({ color: '#B8EEFF', transparent: true, opacity: 0 });
  let lineGroups = [];
  lineGroups[0] = new THREE.Group();
  lineGroups[1] = new THREE.Group();

  // const LinesMaterial = new THREE.LineBasicMaterial({ color: '#B8EEFF', transparent: true, opacity: 0 });
  async function updateLines(arrayName) {

    const previousMaterial = materials[currentMaterialIndex];
    const previousLineGroup = lineGroups[currentMaterialIndex];
    currentMaterialIndex = (currentMaterialIndex + 1) % 2;
    const currentMaterial = materials[currentMaterialIndex];
    const currentLineGroup = lineGroups[currentMaterialIndex];

    if (previousLineGroup.children.length > 0) {
        fadeOut(previousMaterial);
    }
    
    let array = arrays[arrayName];
    zodiacWiki = array[0];
    selectedArray = removeDuplicates(array);
    centerKoords = array[1];
    centerAvailable = true;
    
    await customLookAtControls(centerKoords.y, centerKoords.z, centerKoords.x);

    if (array) {
      for (let i = 2; i < array.length - 1; i++) {
        addLine(array[i], array[i + 1], currentMaterial, currentLineGroup);
      }
      scene.add(currentLineGroup);
      fadeIn(currentMaterial);    
      previousLineGroup.clear();
    scene.remove(previousLineGroup);
    } else {
      console.error("Unbekanntes Array:", arrayName);
    }
  }
  
  function addLine(start, end, material, currentLineGroup) {
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
    let line = new THREE.Line(geometry, material);
    currentLineGroup.add(line);
  }

  function fadeIn(material) {
            return new Promise((resolve) => {
                new TWEEN.Tween(material)
                    .to({ opacity: 1 }, 2000)
                    .easing(TWEEN.Easing.Quadratic.Out)
                    .onComplete(resolve)
                    .start();
            });
  }

  function fadeOut(material) {
            // return new Promise((resolve) => {
                new TWEEN.Tween(material)
                    .to({ opacity: 0 }, 500)
                    .easing(TWEEN.Easing.Quadratic.Out)
                    // .onComplete(resolve)
                    .start();
            // });
  }

  function moveToConstellation(newTarget) {
    gsap.to(controls.target, {
      x: newTarget.x,
      y: newTarget.y,
      z: newTarget.z,
      duration: 2,
      onUpdate: function () {
        // Nach jedem Update der Position muss controls.update() aufgerufen werden, um die Änderungen anzuwenden
        controls.update();
      },
    });
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
    // togglePov();
    toggleValue = false;
    if (centerAvailable)
      customLookAtControls(centerKoords.y, centerKoords.z, centerKoords.x);
    lineGroups[0].clear();
    lineGroups[1].clear();
    centerAvailable = false;
  }

  let showZodiacSelection = false;

  function toggleZodiacSelectionContainer() {
    showZodiacSelection = !showZodiacSelection;
    if(showZodiacSelection) fadeIn(darkZodiacMaterial);
    if (!showZodiacSelection) {
      fadeOut(darkZodiacMaterial);
      showzodiacInfosBox = false;
      resetTest();
    }
  }

  let infoContent = ""; // Inhalt für das zodiacInfosBox
  let showzodiacInfosBox = false;
  let showInfoBox = false; //steuert die Anzeige der lookAtStarInfoBox

  function showInfo(element) {
    // selectedArray = element;
    toggleValue = false;
    updateLines(element);
    infoContent = element;
    showzodiacInfosBox = true;
    showInfoBox = false;
    // displaySmallBox = "block"; // Zeigt das zodiacInfosBox an
    if (!showZodiacSelection) {
      showZodiacSelection = true;
    }
    // console.log("show info");
  }

  function hideInfo() {
    // displaySmallBox = "none"; // Verbirgt das zodiacInfosBox
    showzodiacInfosBox = false;
    // console.log("hide info");
  }

  function adjustCoordinatesForSceneRotation(x, y, z) {
    const angle = THREE.MathUtils.degToRad(337.5); // Negative für Drehung gegen den Uhrzeigersinn
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);

    // Anwenden der Drehung
    const newX = x * cos - y * sin;
    const newY = x * sin + y * cos;
    const newZ = z; // Z bleibt unverändert

    return { x: newX, y: newY, z: newZ };
  }

  async function customLookAt(x, y, z) {
    let targetCoordinates = adjustCoordinatesForSceneRotation(x, y, z);
    const controlPosition = controls.target;
    // console.log(CameraPosition);
    // console.log("Kameraposition:", camera?.position.y);
    // console.log("Ziel der OrbitControls:", controls?.target);

    let dirVector = {
      x: targetCoordinates.x - controlPosition.x,
      y: targetCoordinates.y - controlPosition.y,
      z: targetCoordinates.z - controlPosition.z,
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
    await gsap.to(camera.position, {
      x: newPoint2.x,
      y: newPoint2.y,
      z: newPoint2.z,
      duration: 2,
    });
    // camera.position.set(newPoint2.x, newPoint2.y, newPoint2.z);
  }

  function changeToPov() {
    const cameraPosition = camera.position;
    const controlPosition = controls.target;
    // console.log("Kameraposition:", camera?.position);
    // console.log("Ziel der OrbitControls:", controls?.target);

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

    // console.log(newPoint1);
    moveToConstellation(
      new THREE.Vector3(newPoint1.x, newPoint1.y, newPoint1.z)
    );
  }

  function togglePov() {
    //false == Pov-pov && true == orbit
    toggleValue = !toggleValue;
    if (toggleValue === true) {
      // console.log("using  Orbit");
      let targetCoordinates = adjustCoordinatesForSceneRotation(
        centerKoords.y,
        centerKoords.z,
        centerKoords.x
      );
      moveToConstellation(
        new THREE.Vector3(
          targetCoordinates.x,
          targetCoordinates.y,
          targetCoordinates.z
        )
      );
    } else {
      // console.log("using POV");
      changeToPov();
    }
  }
  let name;

  async function submit() {
  const sce = scene.children.reverse();
  const lowerName = name.toLowerCase();
  name = "Suche nach Stern";
  const starResult = await sce.find((child) => {
    if (child.type === "Mesh") {
      let flamConCombined;
      let bayerConCombined;
      if (child.userData.starData.flam && child.userData.starData.con) {
        const flam = child.userData.starData.flam.toString();
        const lowerCon = child.userData.starData.con.toLowerCase();
        flamConCombined = `${flam} ${lowerCon}`;
      }
      if (child.userData.starData.bayer && child.userData.starData.con) {
        const bayer = child.userData.starData.bayer.toString().toLowerCase();
        const lowerCon = child.userData.starData.con.toLowerCase();
        bayerConCombined = `${bayer} ${lowerCon}`;
      }
      const starName = child.userData.starData.proper?.toLowerCase();
      return child.userData.starData && (starName == lowerName || lowerName === flamConCombined || lowerName === bayerConCombined);
    }
  });
  if (starResult && starResult.userData && starResult.userData.starData) {
    selectedStar = {
      ...starResult.userData.starData,
      object: starResult,
    };
    customLookAtControls(selectedStar.y, selectedStar.z, selectedStar.x);
    showInfoBox = true; // Info-Box anzeigen, wenn ein Stern angeklickt wird
    showzodiacInfosBox = false;
    showSearch = false;
    name = null;
  } else {
    console.log('Kein passender Stern gefunden.');
    name = null;
    // Zusätzliche Logik hier, wenn kein Stern gefunden wurde
  }
}

  function handleKeyPress(event) {
    if (event.keyCode === 13) {
      submit();
    }
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
  import ErdeMitAuge from "./assets/Erde_mit_Auge.svg";
  import constellationX from "./assets/constelX.svg";
  import { draw } from "svelte/transition";
  import { clearcoatRoughness } from "three/examples/jsm/nodes/Nodes.js";

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
    : selectedStar.bayer
    ? `Bayer: ${selectedStar.bayer} ${selectedStar.con}`
    : selectedStar.flam
    ? `Flamsteed: ${selectedStar.flam} ${selectedStar.con}`
    : `HIP: ${selectedStar.hip}`;
  // lastRemovedStar
$: currentHeaderLastRemoved = lastRemovedStar?.proper || lastRemovedStar?.object.userData.starData.proper
  ? `Name: ${lastRemovedStar?.proper || lastRemovedStar?.object.userData.starData.proper}`
  : lastRemovedStar?.bayer || lastRemovedStar?.object.userData.starData.bayer
  ? `Bayer: ${lastRemovedStar?.bayer || lastRemovedStar?.object.userData.starData.bayer} ${lastRemovedStar?.con || lastRemovedStar?.object.userData.starData.con}`
  : lastRemovedStar?.flam || lastRemovedStar?.object.userData.starData.flam
  ? `Flamsteed: ${lastRemovedStar?.flam || lastRemovedStar?.object.userData.starData.flam} ${lastRemovedStar?.con || lastRemovedStar?.object.userData.starData.con}`
  : `HIP: ${lastRemovedStar?.hip || lastRemovedStar?.object.userData.starData.hip}`;


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

  async function customLookAtControls(x, y, z) {
    let targetCoordinates = adjustCoordinatesForSceneRotation(x, y, z);
    const controlPosition = camera.position;
    // console.log(CameraPosition);
    // console.log("Kameraposition:", camera?.position);
    // console.log("Ziel der OrbitControls:", controls?.target);

    let dirVector = {
      x: targetCoordinates.x - controlPosition.x,
      y: targetCoordinates.y - controlPosition.y,
      z: targetCoordinates.z - controlPosition.z,
    };

    let length = Math.sqrt(
      dirVector.x ** 2 + dirVector.y ** 2 + dirVector.z ** 2
    );

    dirVector.x /= length;
    dirVector.y /= length;
    dirVector.z /= length;

    const distance = 0.01;

    let newPoint2 = {
      x: controlPosition.x + dirVector.x * distance,
      y: controlPosition.y + dirVector.y * distance,
      z: controlPosition.z + dirVector.z * distance,
    };
    console.log(newPoint2);
    gsap.to(controls.target, {
      x: newPoint2.x,
      y: newPoint2.y,
      z: newPoint2.z,
      duration: 1,
      onUpdate: function () {
        controls.update();
      },
    });
  }

  async function lookAtSun() {
    const center = new THREE.Vector3(0, 0, 0); // Zentralpunkt, der betrachtet wird
    const distance = camera.position.distanceTo(center); // Distanz von der Kamera zum Zentrum
    const sce = scene.children.reverse();
    const sun = sce.find((child) => {
      if (child.type === "Mesh") {
        // console.log("-");
        if (child.userData.starData.id === 1) console.log("hit");
        return child.userData.starData && child.userData.starData.id === 1;
      }
    });

    if(sun) {
      const originalColor = sun.material.color.clone();
      const originalEmissive = sun.material.emissive.clone();

      sun.material.color.set(0xff0000);
      sun.material.emissive.set(0xff0000);
      
      setTimeout(() => {
          sun.material.color.copy(originalColor);
          sun.material.emissive.copy(originalEmissive);
      }, 500);
    }

    if (distance > 1 && sun === undefined) {
      const Sonne = {
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
      addStars([Sonne]);
      lastRemovedStar = null;
    } else if (distance > 1 && sun) {
      // console.log(
      //   "Kamera befindet sich außerhalb des Radius von 10 Einheiten um das Zentrum, aber Sonne existiert bereits."
      // );
      // Optional: Führe eine andere Aktion aus oder mache nichts
    } else {
      // console.log(
      //   "Kamera befindet sich innerhalb des Radius von 10 Einheiten um das Zentrum"
      // );
    }
    customLookAtControls(0, 0, 0);
  }
</script>

<!-- HTML -->
<main>
  <div id="tooltip" >Tooltip</div>

  <div
    id="zodiacInfosBox"
    class="overlay"
    style="display: {showzodiacInfosBox ? 'flex' : 'none'} !important;"
  >
    <!-- Stellen Sie sicher, dass alle Inhalte hier innerhalb sind -->
    <div class="zodiacStarLinksHeader" id="zodiacInfosBoxName">
      {infoContent}
      <button
        id="wikiLinkButton"
        on:click|stopPropagation={() => window.open(zodiacWiki, "_blank")}
        >Wikipedia</button
      >
    </div>
    <div class="zodiacContent">
      <!-- Diese Klasse dient zur Abgrenzung des Inhaltsbereichs -->
      {#if selectedArray && selectedArray.length > 0}
        {#each selectedArray as star}
          <button
            class="jumpToStar2Button"
            on:click|stopPropagation={() => jumpToStar2(star)}
          >
            {star.proper ? star.proper : star.flam + star.con || star.hip}
          </button>
        {/each}
      {/if}
    </div>
  </div>

  <div
    id="zodiacSelectionContainer"
    style="--translateX: {showZodiacSelection ? '0%' : '-100%'};"
  >
    <div id="zodiacSelectionOverlay">
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Steinbock")}
        ><img class="svgIcon" src={Steinbock} alt="Steinbock" />
        <span>Steinbock</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Wassermann")}
        ><img class="svgIcon" src={Wassermann} alt="Wassermann" />
        <span>Wassermann</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Fische")}
        ><img class="svgIcon" src={Fische} alt="Fische" />
        <span>Fische</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Widder")}
        ><img class="svgIcon" src={Widder} alt="Widder" />
        <span>Widder</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Stier")}
        ><img class="svgIcon" src={Stier} alt="Stier" /> <span>Stier</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Zwilling")}
        ><img class="svgIcon" src={Zwillinge} alt="Zwilling" />
        <span>Zwilling</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Krebs")}
        ><img class="svgIcon" src={Krebs} alt="Krebs" /> <span>Krebs</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Löwe")}
        ><img class="svgIcon" src={Löwe} alt="Löwe" /> <span>Löwe</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Jungfrau")}
        ><img class="svgIcon" src={Jungfrau} alt="Jungfrau" />
        <span>Jungfrau</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Waage")}
        ><img class="svgIcon" src={Waage} alt="Waage" /> <span>Waage</span>
      </button>
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Skorpion")}
        ><img class="svgIcon" src={Skorpion} alt="Skorpion" />
        <span>Skorpion</span>
      </button>
      <!-- schlangenträger -->
      <button
        class="zodiacButtons"
        on:click|stopPropagation={() => showInfo("Schütze")}
        ><img class="svgIcon" src={Schütze} alt="Schütze" />
        <span>Schhütze</span>
      </button>
      <!-- more -->
      <!-- ansicht in raster -->
    </div>
    <!-- <button class="quickSelectButtons" on:click={() => resetTest()}
      ><img class="svgIcon" src={constellation} alt="reset to sun" /></button
    > -->
    <div
      id="searchBar"
      bind:this={searchRef}
      style="display: {showSearch ? 'flex' : 'none'};"
    >
    <input bind:value={name} placeholder="enter name of star" on:keyup={handleKeyPress} />
      <button on:click|stopPropagation={submit}>Submit</button>
    </div>
    <!-- quickBar -->
    <div id="quickSelectBar">
      <button
        class="quickSelectButtons"
        on:click={toggleZodiacSelectionContainer}
      >
        {#if showZodiacSelection}
          <img
            class="quickSelectIcons"
            src={constellationX}
            alt="Toggle visibility"
          />
        {:else}
          <img
            class="quickSelectIcons"
            src={constellation}
            alt="Toggle visibility"
          />
        {/if}
      </button>
      <button class="quickSelectButtons" on:click|stopPropagation={returnToSun}>
        <img class="quickSelectIcons" src={Erde} alt="Return to Sun" />
      </button>
      <button class="quickSelectButtons" on:click|stopPropagation={lookAtSun}>
        <!-- <button class="quickSelectButtons" on:click|stopPropagation={() => customLookAtControls(0,0,0)}> -->
        <img class="quickSelectIcons" src={ErdeMitAuge} alt="Look at Sun" />
      </button>
      <button
        class="quickSelectButtons"
        on:click|stopPropagation={toggleSearch}
      >
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
    <p>Entfernung: {(selectedStar.dist * 3.26).toFixed(2)} Lichtjahre</p>
    <p>Farbindex: {selectedStar.ci}</p>
    <p>AbsoluteMagnitude: {selectedStar.absmag}</p>

    <button id="jumpButton" on:click|stopPropagation={jumpToStar}>Sprung</button
    >
    <button
      id="wikiLinkButton"
      on:click|stopPropagation={() =>
        window.open(selectedStar.wikiUrl, "_blank")}>Wikipedia</button
    >
  </div>

  <div
    class="infoBoxPosition"
    style="display: {showCurrentInfosBox ? 'block' : 'none'} !important;"
  >
    <div class="currentPosition">
      {currentHeaderLastRemoved}
    </div>
    <div style="display: flex; flex-direction: row; ">
      <div style="display: flex;
    flex-direction: column;
    align-items: start; ">
    <p style="margin-bottom: 5px;">Magnitude: {lastRemovedStar.mag}</p>
        <p style="margin-bottom: 5px;">Entfernung: {(lastRemovedStar.dist * 3.26).toFixed(2)} Lichtjahre</p>
      </div>
      <div style="margin-left:20px; display: flex;
    flex-direction: column;
    align-items: start;">
    <p style="margin-bottom: 5px;">Farbindex: {lastRemovedStar.ci}</p>
        <p style="margin-bottom: 5px;">AbsoluteMagnitude: {lastRemovedStar.absmag}</p>
      </div>
    </div>
  </div>

  <div
    id="togglePovButton"
    style="display: {centerAvailable ? 'block' : 'none'}"
  >
    <button id="tglbtn" on:click|stopPropagation={togglePov}>
      {toggleValue ? "Orbit" : "POV"}
    </button>
  </div>
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

  #tglbtn {
    background-color: transparent;
    padding: 20px;
    border: none;
    font-size: 22px;
  }

  #tglbtn:hover {
    background-color: rgba(255, 255, 255, 0.2);
  }

  #togglePovButton {
    position: absolute;
    top: 0;
    height: 70px;
    width: 100px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 100;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    background-color: rgba(166, 166, 166, 0.093);
    border-radius: 0;
    border-bottom-right-radius: 8px;
    border-bottom-left-radius: 8px;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }

  #quickSelectBar {
    padding-left: 10px;
    padding-right: 10px;
    background-color: rgba(166, 166, 166, 0.093);
    color: white;
    max-height: 70px;
    /* border-bottom-right-radius: 10px; */
    margin: 0;
    display: flex;
    /* gap: 10px; */
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    /* border-top-right-radius: 8px; */
    border-bottom-right-radius: 8px;
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
    top: 82px;
    left: 10px;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
  }

  #searchBar input {
    padding: 10px;
    border: 2px solid #b7b7b7;
    border-radius: 8px;
    margin-right: 5px;
    background-color: rgba(166, 166, 166, 0.093);
    border-bottom-right-radius: 8px;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }

  #searchBar button {
    padding: 10px 20px;
    background-color: #8a8a8a;
    border: none;
    border-radius: 8px;
    color: #fff;
    cursor: pointer;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    background-color: rgba(166, 166, 166, 0.093);
    border-bottom-right-radius: 8px;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }

  .zodiacButtons {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 10px 5px;
    background-color: transparent;
    width: 100%;
    border: none;
    color: white;
    text-align: center;
    cursor: pointer;
    transition: background-color 0.3s ease;
    position: relative;
  }

  :focus {
    outline: none;
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

  #wikiLinkButton:hover {
    background-color: rgba(255, 255, 255, 0.2);
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
    bottom: 2px;
    left: 50%;
    transform: translate(-50%);
    background-color: rgba(166, 166, 166, 0.093);
    padding: 20px;
    padding-top: 6px;
    padding-bottom: 6px;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    /* border-radius: 8px; */
    border-top-left-radius: 8px;
    border-top-right-radius: 8px;
    cursor: default;
    z-index: 100;
    min-height: 50px;
    min-width: 200px;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
  }

  .currentPosition {
    font-size: 24px;
    font-weight: bold;
    padding: 10px;
    padding-bottom: initial;
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
    left: 140px;
    transform: translateX(var(--translateX));
    transition: transform 0.5s ease;
  }

  #zodiacSelectionOverlay {
    position: absolute;
    display: flex;
    flex-direction: column;
    background-color: rgba(166, 166, 166, 0.093);
    padding: 10px;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    width: 80%;
    left: -142px;
    backdrop-filter: blur(4px);
    border-bottom-right-radius: 8px;
    -webkit-backdrop-filter: blur(4px);
    max-height: 97vh;
    overflow-y: auto;
  }

  #zodiacSelectionOverlay::-webkit-scrollbar {
    display: none; /* Für WebKit-Browser wie Chrome und Safari */
  }
  /* ende sternzeichen auswahl Kontainer ende */

  #zodiacInfosBox::-webkit-scrollbar {
    display: none; /* Für WebKit-Browser wie Chrome und Safari */
  }

  #zodiacInfosBoxName {
    position: sticky;
    top: 0; /* Hier stellen Sie ein, dass es am oberen Rand "kleben" bleibt */
    color: white;
    font-size: 28px;
    z-index: 1; /* Stellt sicher, dass die Überschrift über den anderen Inhalten liegt */
    display: flex;
    flex-direction: column;
  }

  #zodiacInfosBox {
    position: absolute;
    top: 10px;
    right: 10px;
    background-color: rgba(166, 166, 166, 0.093);
    padding: 10px;
    padding-left: 20px;
    padding-right: 20px;
    padding-bottom: 0px;
    max-height: 47vh;
    overflow-y: auto;
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    border-radius: 8px;
    min-width: 220px;
    /* cursor: pointer; */
    border: 0.5px solid rgba(72, 72, 72, 0.248);
    text-align: left; /* Add this line to align the content to the left */
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
    display: flex;
    flex-direction: column;
  }

  .zodiacContent {
    overflow-y: scroll;
    scroll-behavior: smooth;
    display: flex;
    flex-direction: column;
    padding-bottom: 10px;
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
  #tooltip {
  position: absolute;
  display: none;
  background-color: rgba(166, 166, 166, 0.093);
  color: white;
  padding: 5px;
  border-radius: 5px;
  font-size: 14px;
  z-index: 1000;
  pointer-events: none;
  border: 0.5px solid rgba(72, 72, 72, 0.248);
  backdrop-filter: blur(4px);
  -webkit-backdrop-filter: blur(4px);
  transform: translateX(-50%);
}
</style>
