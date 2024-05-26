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
  let counter = 0;

  const leoHip = [
    49669, 57632, 50583, 54872, 54879, 47908, 48455, 49641, 50555, 51069, 55434,
  ];

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
      300
    );
    camera.position.z = 0.0001;
    const angleInDegrees = 23.5;
  const angleInRadians = angleInDegrees * (Math.PI / 180);

// Kamera um die x-Achse rotieren
    camera.rotation.x += angleInRadians;
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
            ra: star.ra,
            dec: star.dec,
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



  function calculateCoordinates(ra, dec, dist) {
    // Konvertiere RA von Stunden in Grad (1 Stunde = 15 Grad)
    let raDeg = ra * 15;

    // Konvertiere RA und DEC in Radianten für die trigonometrischen Funktionen
    let raRad = raDeg * Math.PI / 180;
    let decRad = dec * Math.PI / 180;

    // Berechne die kartesischen Koordinaten
    let x = dist * Math.cos(decRad) * Math.cos(raRad);
    let y = dist * Math.cos(decRad) * Math.sin(raRad);
    let z = dist * Math.sin(decRad);

    return { x, y, z };
}

  function addStars(stars) {
    if (stars.length === 0) {
      console.error("Keine gültigen Sterndaten verfügbar.");
      return;
    }

    // Materialien definieren
const greenMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
const blueMaterial = new THREE.MeshBasicMaterial({ color: 0x0000ff });
const redMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });

// Geometrie definieren
const geometry = new THREE.BoxGeometry(0.1, 0.1, 0.1);

// Quader erstellen
const greenCube = new THREE.Mesh(geometry, greenMaterial);
greenCube.position.set(1, 0, 0);
scene.add(greenCube);

const blueCube = new THREE.Mesh(geometry, blueMaterial);
blueCube.position.set(0, 1, 0);
scene.add(blueCube);

const redCube = new THREE.Mesh(geometry, redMaterial);
redCube.position.set(0, 0, 1);
scene.add(redCube);







    console.log(stars);
    stars.forEach((star) => {

      if (star.id === 1 && sunIgnored == false) {
        sunIgnored = true;
        return;
      }
      let starGeometry;

      let coordinates = calculateCoordinates(star.ra, star.dec, star.dist);

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
        intensity = 0xff0000;
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
      sphere.position.set(coordinates.x, coordinates.z, coordinates.y);
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

  let arrays = {
    leoLines: [
      { x0: -29.062, y0: 18.026, z0: 16.685 },
      { x0: -53.196, y0: 35.259, z0: 28.114 },
      { x0: -21.019, y0: 11.132, z0: 5.041 },
      { x0: -115.275, y0: 61.333, z0: -0.847 },
      { x0: -273.354, y0: 128.03, z0: -431.558 },
      { x0: -65.657, y0: 28.518, z0: -21.662 },
      { x0: -16.442, y0: 3.337, z0: 6.281 },
      { x0: -34.003, y0: 15.861, z0: 13.539 },
      { x0: -21.019, y0: 11.132, z0: 5.041 },
      // { x0: -46.506, y0: 9.411, z0: 13.096 },
      // { x0: -72.313, y0: 12.381, z0: 7.749 },
      // { x0: -10.634, y0: 0.508, z0: 2.768 }
    ],
    fische: [
      { x0: 38.448, y0: 12.814, z0: 5.39 },
      { x0: 31.621, y0: -30.021, z0: -13.525 },
      { x0: 41.496, y0: -7.847, z0: 2.422 },
      { x0: 48.701, y0: -7.076, z0: 1.079 },
      { x0: 25.568, y0: -3.193, z0: -1.841 },
      { x0: 38.777, y0: -3.429, z0: -9.866 },
      { x0: 13.595, y0: -1.192, z0: 1.344 },
      { x0: 31.941, y0: -2.507, z0: 0.996 },
      { x0: 32.792, y0: -0.099, z0: 3.947 },
    ],
    Zwilling: [
       { x0: -0.102, y0: 75.752, z0: -171.951 },
  { x0: -6.562, y0: 65.281, z0: 27.195 },
  { x0: -26.594, y0: 209.318, z0: 77.684 },
  { x0: -5.266, y0: 31.714, z0: 9.461 },
  { x0: -46.024, y0: 237.15, z0: 113.322 },
  { x0: -3.532, y0: 17.638, z0: 4.118 },
  { x0: -11.225, y0: 47.866, z0: 33.114 },
  { x0: -84.119, y0: 292.833, z0: 114.34 },
  { x0: -31.691, y0: 98.796, z0: 60.496 },
  { x0: -9.825, y0: 27.709, z0: 8.731 },
  { x0: -5.888, y0: 16.151, z0: 6.94 },
  { x0: -13.456, y0: 34.281, z0: 19.415 },
  { x0: -5.312, y0: 12.13, z0: 8.239 },
  { x0: -28.15, y0: 63.284, z0: 35.133 },
  { x0: -17.374, y0: 35.447, z0: 17.905 },
  { x0: -4.055, y0: 8.195, z0: 4.867 },
    ],
    Widder: [
      { _id: "663b7e1c2bdcc11befd4a1a5", x0: 41.77, y0: 22.568, z0: 16.62 },
  { _id: "663b7e1c2bdcc11befd4a24c", x0: 14.753, y0: 8.063, z0: 6.389 },
  { _id: "663b7e1c2bdcc11befd4ab7b", x0: 15.732, y0: 9.752, z0: 8.034 },
  { _id: "663b7e1e2bdcc11befd4c94e", x0: 33.288, y0: 30.498, z0: 23.262},
],Wassermann: [
  {
    _id: '663b7e542bdcc11befd943d5',
    hip: 115438,
    ra: 23.382842,
    dec: -20.10058,
    dist: 50.3632,
    x0: 46.68,
    y0: -7.608,
    z0: -17.308
  },
  {
    _id: '663b7e532bdcc11befd93eab',
    hip: 114855,
    ra: 23.26485994,
    dec: -9.08773573,
    dist: 45.2645,
    x0: 43.871,
    y0: -8.549,
    z0: -7.149
  },
  {
    _id: '663b7e522bdcc11befd92d35',
    hip: 112961,
    ra: 22.87691,
    dec: -7.579599,
    dist: 111.9069,
    x0: 106.169,
    y0: -32.148,
    z0: -14.761
  },
  {
    _id: '663b7e522bdcc11befd91fa0',
    hip: 111497,
    ra: 22.589272,
    dec: -0.117498,
    dist: 54.434,
    x0: 50.763,
    y0: -19.65,
    z0: -0.112
  },
  {
    _id: '663b7e522bdcc11befd91a77',
    hip: 110960,
    ra: 22.480531,
    dec: -0.019972,
    dist: 28.169,
    x0: 25.97,
    y0: -10.912,
    z0: -0.01
  },
  {
    _id: '663b7e512bdcc11befd9149b',
    hip: 110395,
    ra: 22.360938,
    dec: -1.387331,
    dist: 50.2008,
    x0: 45.636,
    y0: -20.88,
    z0: -1.215
  },
  {
    _id: '663b7e512bdcc11befd90710',
    hip: 109074,
    ra: 22.09639885,
    dec: -0.31984929,
    dist: 202.2185,
    x0: 177.619,
    y0: -96.656,
    z0: -1.129
  },
  {
    _id: '663b7e4f2bdcc11befd8e928',
    hip: 106278,
    ra: 21.525982,
    dec: -5.571172,
    dist: 167.4261,
    x0: 132.887,
    y0: -100.54,
    z0: -16.254
  },
  {
    _id: '663b7e4e2bdcc11befd8bdc8',
    hip: 102618,
    ra: 20.79459785,
    dec: -9.49577641,
    dist: 63.6943,
    x0: 41.97,
    y0: -46.745,
    z0: -10.508
  },
  {
    _id: '663b7e4f2bdcc11befd8e928',
    hip: 106278,
    ra: 21.525982,
    dec: -5.571172,
    dist: 167.4261,
    x0: 132.887,
    y0: -100.54,
    z0: -16.254
  },
  {
    _id: '663b7e512bdcc11befd90710',
    hip: 109074,
    ra: 22.09639885,
    dec: -0.31984929,
    dist: 202.2185,
    x0: 177.619,
    y0: -96.656,
    z0: -1.129
  },
  {
    _id: '663b7e512bdcc11befd910b1',
    hip: 110003,
    ra: 22.28056584,
    dec: -7.78328831,
    dist: 58.5163,
    x0: 52.202,
    y0: -25.226,
    z0: -7.925
  },
  {
    _id: '663b7e512bdcc11befd907a2',
    hip: 109139,
    ra: 22.10728585,
    dec: -13.86968033,
    dist: 64.5411,
    x0: 55.123,
    y0: -29.793,
    z0: -15.471
  },
  {
    _id: '663b7e512bdcc11befd910b1',
    hip: 110003,
    ra: 22.28056584,
    dec: -7.78328831,
    dist: 58.5163,
    x0: 52.202,
    y0: -25.226,
    z0: -7.925
  },
  {
    _id: '663b7e522bdcc11befd91c07',
    hip: 111123,
    ra: 22.51078213,
    dec: -10.67796039,
    dist: 88.8099,
    x0: 80.723,
    y0: -33.17,
    z0: -16.455
  },
  {
    _id: '663b7e522bdcc11befd92aeb',
    hip: 112716,
    ra: 22.82652815,
    dec: -13.59262957,
    dist: 99.8288,
    x0: 92.49,
    y0: -29.343,
    z0: -23.461
  },
  {
    _id: '663b7e532bdcc11befd92ea6',
    hip: 113136,
    ra: 22.910837,
    dec: -15.82082,
    dist: 49.2368,
    x0: 45.459,
    y0: -13.325,
    z0: -13.423
  },
  {
    _id: '663b7e532bdcc11befd939fd',
    hip: 114341,
    ra: 23.157443,
    dec: -21.17241,
    dist: 78.9203,
    x0: 71.81,
    y0: -16.102,
    z0: -28.504
  },
],

};

let selectedArray;
$: if (selectedArray) {
  updateLines(selectedArray);
  console.log("debug");   
}

function updateLines(arrayName) {
  // scene.clear(); // Bereinigt die aktuelle Szene, um eine neue Linienkonfiguration zu zeichnen
    console.log(arrayName);
    let array = arrays[arrayName];
    if (array) {
      for (let i = 0; i < array.length - 1; i++) {
        addLine(array[i], array[i + 1]);
      }
    } else {
      console.error("Unbekanntes Array:", arrayName);
    }
  }

  function addLine(start, end) {
    let geometry = new THREE.BufferGeometry();
    let vertices = new Float32Array([
      start.y0,
      start.z0,
      start.x0,
      end.y0,
      end.z0,
      end.x0,
    ]);
    geometry.setAttribute("position", new THREE.BufferAttribute(vertices, 3));
    let color = Math.floor(Math.random() * 0xffffff);
    let material = new THREE.LineBasicMaterial({ color: color });
    let line = new THREE.Line(geometry, material);
    scene.add(line);
  }
</script>

<main>
  <select bind:value={selectedArray}>
    {#each Object.keys(arrays) as arrayName}
      <option value={arrayName}>{arrayName}</option>
    {/each}
  </select>

  <!-- <button on:click={connectStars}>Connect Leo</button> -->
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
