<script>
  import { onMount } from "svelte";
  import axios from "axios";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
  import gsap from "gsap";

  let raycaster = new THREE.Raycaster();
  let mouse = new THREE.Vector2();

  let camera, scene, renderer, controls;
  const maxMag = 8;
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
    );
    camera.position.z = 0.0001;

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000000); // Setzen Sie explizit eine Hintergrundfarbe

    scene.rotation.z = THREE.MathUtils.degToRad(337.5);

    renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement); // Stellen Sie sicher, dass dies ausgeführt wird
    controls = new OrbitControls(camera, renderer.domElement);
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
          }));
        addStars(starsData);
        animate();
      })
      .catch((error) => {
        console.error("Fehler beim Abrufen der Sterndaten:", error);
      });
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
      sphere.position.set(star.y, star.z, star.x);
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
      console.log("click registered");
      hideInfo();
      let firstObject = intersects[0].object;
      if (firstObject.userData.starData) {
        console.log(firstObject.userData.starData);

        if (lastRemovedStar != null) addStars([lastRemovedStar]);

        lastRemovedStar = null;

        selectedStar = {
          ...firstObject.userData.starData,
          object: firstObject,
        };
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
    // const hyperspaceLines = createHyperspaceLines();
    // console.log(hyperspaceLines);
    // animateHyperspaceLines(hyperspaceLines);

    await gsap.to(camera.position, {
      x: newPosition.x,
      y: newPosition.y,
      z: newPosition.z,
      duration: 2,
    });

    if (selectedStar.object) {
      scene.remove(selectedStar.object);
      disposeMaterial(selectedStar.object);
    }

    // Aktualisieren von lastRemovedStar
    lastRemovedStar = selectedStar;
  }

  function returnToSun() {
    hideInfo();
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

  let arrays = {
    leoLines: [
      { x: -69.8918888888889, y: 34.736444444444444, z: -42.151777777777774 },
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
    Zwilling: [
      { x: -18.741125000000004, y: 77.5919375, z: 34.9838125 },
      { x0: -3.532, y0: 17.638, z0: 4.118 },
      { x0: -9.825, y0: 27.709, z0: 8.731 },
      { x0: -5.888, y0: 16.151, z0: 6.94 },
      { x0: -84.119, y0: 292.833, z0: 114.34 },
      { x0: -5.266, y0: 31.714, z0: 9.461 },
      { x0: -84.119, y0: 292.833, z0: 114.34 },
      { x0: -5.888, y0: 16.151, z0: 6.94 },
      { x0: -28.15, y0: 63.284, z0: 35.133 },
      { x0: -17.374, y0: 35.447, z0: 17.905 },
      { x0: -28.15, y0: 63.284, z0: 35.133 },
      { x0: -4.055, y0: 8.195, z0: 4.867 },
      { x0: -28.15, y0: 63.284, z0: 35.133 },
      { x0: -13.456, y0: 34.281, z0: 19.415 },
      { x0: -31.691, y0: 98.796, z0: 60.496 },
      { x0: -5.312, y0: 12.13, z0: 8.239 },
      { x0: -31.691, y0: 98.796, z0: 60.496 },
      { x0: -11.225, y0: 47.866, z0: 33.114 },
      { x0: -31.691, y0: 98.796, z0: 60.496 },
      { x0: -46.024, y0: 237.15, z0: 113.322 },
      { x0: -26.594, y0: 209.318, z0: 77.684 },
      { x0: -46.024, y0: 237.15, z0: 113.322 },
      { x0: -6.562, y0: 65.281, z0: 27.195 },
      { x0: -6.562, y0: 65.281, z0: 27.195 },
      { x0: -0.785, y0: 43.678, z0: 18.781 },
    ],
    Widder: [
      { x: 26.385749999999998, y: 17.72025, z: 13.57625 },
      { _id: "663b7e1c2bdcc11befd4a1a5", x0: 41.77, y0: 22.568, z0: 16.62 },
      { _id: "663b7e1c2bdcc11befd4a24c", x0: 14.753, y0: 8.063, z0: 6.389 },
      { _id: "663b7e1c2bdcc11befd4ab7b", x0: 15.732, y0: 9.752, z0: 8.034 },
      { _id: "663b7e1e2bdcc11befd4c94e", x0: 33.288, y0: 30.498, z0: 23.262 },
    ],
    Wassermann: [
      {
        x: 71.29146666666665,
        y: -32.709799999999994,
        z: -11.578999999999999,
      },
      {
        _id: "663b7e542bdcc11befd943d5",
        hip: 115438,
        ra: 23.382842,
        dec: -20.10058,
        dist: 50.3632,
        x0: 46.68,
        y0: -7.608,
        z0: -17.308,
      },
      {
        _id: "663b7e532bdcc11befd93eab",
        hip: 114855,
        ra: 23.26485994,
        dec: -9.08773573,
        dist: 45.2645,
        x0: 43.871,
        y0: -8.549,
        z0: -7.149,
      },
      {
        _id: "663b7e522bdcc11befd92d35",
        hip: 112961,
        ra: 22.87691,
        dec: -7.579599,
        dist: 111.9069,
        x0: 106.169,
        y0: -32.148,
        z0: -14.761,
      },
      {
        _id: "663b7e522bdcc11befd91fa0",
        hip: 111497,
        ra: 22.589272,
        dec: -0.117498,
        dist: 54.434,
        x0: 50.763,
        y0: -19.65,
        z0: -0.112,
      },
      {
        _id: "663b7e522bdcc11befd91a77",
        hip: 110960,
        ra: 22.480531,
        dec: -0.019972,
        dist: 28.169,
        x0: 25.97,
        y0: -10.912,
        z0: -0.01,
      },
      {
        _id: "663b7e512bdcc11befd9149b",
        hip: 110395,
        ra: 22.360938,
        dec: -1.387331,
        dist: 50.2008,
        x0: 45.636,
        y0: -20.88,
        z0: -1.215,
      },
      {
        _id: "663b7e512bdcc11befd90710",
        hip: 109074,
        ra: 22.09639885,
        dec: -0.31984929,
        dist: 202.2185,
        x0: 177.619,
        y0: -96.656,
        z0: -1.129,
      },
      {
        _id: "663b7e4f2bdcc11befd8e928",
        hip: 106278,
        ra: 21.525982,
        dec: -5.571172,
        dist: 167.4261,
        x0: 132.887,
        y0: -100.54,
        z0: -16.254,
      },
      {
        _id: "663b7e4e2bdcc11befd8bdc8",
        hip: 102618,
        ra: 20.79459785,
        dec: -9.49577641,
        dist: 63.6943,
        x0: 41.97,
        y0: -46.745,
        z0: -10.508,
      },
      {
        _id: "663b7e4f2bdcc11befd8e928",
        hip: 106278,
        ra: 21.525982,
        dec: -5.571172,
        dist: 167.4261,
        x0: 132.887,
        y0: -100.54,
        z0: -16.254,
      },
      {
        _id: "663b7e512bdcc11befd90710",
        hip: 109074,
        ra: 22.09639885,
        dec: -0.31984929,
        dist: 202.2185,
        x0: 177.619,
        y0: -96.656,
        z0: -1.129,
      },
      {
        _id: "663b7e512bdcc11befd910b1",
        hip: 110003,
        ra: 22.28056584,
        dec: -7.78328831,
        dist: 58.5163,
        x0: 52.202,
        y0: -25.226,
        z0: -7.925,
      },
      {
        _id: "663b7e512bdcc11befd907a2",
        hip: 109139,
        ra: 22.10728585,
        dec: -13.86968033,
        dist: 64.5411,
        x0: 55.123,
        y0: -29.793,
        z0: -15.471,
      },
      {
        _id: "663b7e512bdcc11befd910b1",
        hip: 110003,
        ra: 22.28056584,
        dec: -7.78328831,
        dist: 58.5163,
        x0: 52.202,
        y0: -25.226,
        z0: -7.925,
      },
      {
        _id: "663b7e522bdcc11befd91c07",
        hip: 111123,
        ra: 22.51078213,
        dec: -10.67796039,
        dist: 88.8099,
        x0: 80.723,
        y0: -33.17,
        z0: -16.455,
      },
      {
        _id: "663b7e522bdcc11befd92aeb",
        hip: 112716,
        ra: 22.82652815,
        dec: -13.59262957,
        dist: 99.8288,
        x0: 92.49,
        y0: -29.343,
        z0: -23.461,
      },
      {
        _id: "663b7e532bdcc11befd92ea6",
        hip: 113136,
        ra: 22.910837,
        dec: -15.82082,
        dist: 49.2368,
        x0: 45.459,
        y0: -13.325,
        z0: -13.423,
      },
      {
        _id: "663b7e532bdcc11befd939fd",
        hip: 114341,
        ra: 23.157443,
        dec: -21.17241,
        dist: 78.9203,
        x0: 71.81,
        y0: -16.102,
        z0: -28.504,
      },
    ],
    Skorpion: [
      { x: -29.960153846153844, y: -122.941, z: -88.8423076923077 },
      { x0: -16.049, y0: -138.75, z0: -105.65 },
      { x0: -8.785, y0: -114.748, z0: -93.293 },
      { x0: -29.202, y0: -538.527, z0: -454.583 },
      { x0: -6.654, y0: -67.017, z0: -62.797 },
      { x0: -3.401, y0: -16.055, z0: -15.432 },
      { x0: -8.585, y0: -29.255, z0: -27.803 },
      { x0: -35.433, y0: -115.662, z0: -94.672 },
      { x0: -4.842, y0: -15.396, z0: -11.007 },
      { x0: -45.96, y0: -119.547, z0: -68.721 },
      { x0: -58.543, y0: -140.308, z0: -75.575 },
      { x0: -46.814, y0: -80.155, z0: -45.503 },
      { x0: -58.543, y0: -140.308, z0: -75.575 },
      { x0: -69.333, y0: -120.492, z0: -57.928 },
      { x0: -58.543, y0: -140.308, z0: -75.575 },
      { x0: -55.881, y0: -102.321, z0: -41.986 },
    ],
    Fische: [
      { x: 74.4677777777778, y: 19.51683333333333, z: 16.47005555555556 },
      { x0: 105.454, y0: 35.156, z0: 50.855 },
      { x0: 96.78, y0: 27.212, z0: 62.343 },
      { x0: 82.363, y0: 29.761, z0: 45.131 },
      { x0: 105.454, y0: 35.156, z0: 50.855 },
      { x0: 101.738, y0: 42.915, z0: 30.302 },
      { x0: 75.809, y0: 37.547, z0: 13.638 },
      // stern daneben
      { x0: 41.361, y0: 24.472, z0: 2.364 },
      { x0: 48.236, y0: 26.069, z0: 3.054 },
      { x0: 100.757, y0: 47.752, z0: 10.712 },
      { x0: 98.543, y0: 40.911, z0: 11.485 },
      { x0: 53.54, y0: 15.086, z0: 7.709 },
      { x0: 176.504, y0: 37.751, z0: 23.122 },
      { x0: 139.669, y0: 12.587, z0: 20.184 },
      { x0: 32.792, y0: -0.099, z0: 3.947 },
      { x0: 13.595, y0: -1.192, z0: 1.344 },
      { x0: 51.141, y0: -7.195, z0: 5.774 },
      { x0: 41.496, y0: -7.847, z0: 2.422 },
      { x0: 48.701, y0: -7.076, z0: 1.079 },
      { x0: 31.941, y0: -2.507, z0: 0.996 },
      { x0: 13.595, y0: -1.192, z0: 1.344 },
    ],
    taurus: [
      {
        _id: "663b7e1f2bdcc11befd4e074",
        hip: 15900,
        ra: 3.4135552,
        dec: 9.02887505,
        dist: 89.2061,
        x0: 55.2,
        y0: 68.663,
        z0: 13.999,
      },
      {
        _id: "663b7e202bdcc11befd4f8e6",
        hip: 18724,
        ra: 4.01133764,
        dec: 12.49034684,
        dist: 124.3823,
        x0: 60.407,
        y0: 105.349,
        z0: 26.901,
      },
      {
        _id: "663b7e212bdcc11befd50633",
        hip: 20205,
        ra: 4.32988998,
        dec: 15.6276446,
        dist: 46.1584,
        x0: 18.823,
        y0: 40.27,
        z0: 12.434,
      },
      {
        _id: "663b7e212bdcc11befd50c8a",
        hip: 20894,
        ra: 4.4777058,
        dec: 15.87088179,
        dist: 46.1042,
        x0: 17.21,
        y0: 40.872,
        z0: 12.608,
      },

      {
        _id: "663b7e212bdcc11befd51178",
        hip: 21421,
        ra: 4.5986668,
        dec: 16.50976164,
        dist: 20.4332,
        x0: 7.027,
        y0: 18.288,
        z0: 5.807,
      },
      {
        _id: "663b7e242bdcc11befd547f6",
        hip: 26451,
        ra: 5.627413,
        dec: 21.142549,
        dist: 136.4256,
        x0: 12.392,
        y0: 126.638,
        z0: 49.207,
      },
      {
        _id: "663b7e212bdcc11befd51178",
        hip: 21421,
        ra: 4.5986668,
        dec: 16.50976164,
        dist: 20.4332,
        x0: 7.027,
        y0: 18.288,
        z0: 5.807,
      },
      {
        _id: "663b7e212bdcc11befd50c7e",
        hip: 20889,
        ra: 4.47694409,
        dec: 19.18043252,
        dist: 44.712,
        x0: 16.396,
        y0: 38.917,
        z0: 14.69,
      },
      {
        _id: "663b7e222bdcc11befd515d4",
        hip: 21881,
        ra: 4.70408391,
        dec: 22.95694022,
        dist: 122.1001,
        x0: 37.416,
        y0: 106.021,
        z0: 47.624,
      },
      {
        _id: "663b7e242bdcc11befd53bb7",
        hip: 25428,
        ra: 5.43819387,
        dec: 28.60787346,
        dist: 41.0509,
        x0: 5.282,
        y0: 35.65,
        z0: 19.655,
      },
      {
        _id: "663b7e222bdcc11befd515d4",
        hip: 21881,
        ra: 4.70408391,
        dec: 22.95694022,
        dist: 122.1001,
        x0: 37.416,
        y0: 106.021,
        z0: 47.624,
      },
      {
        _id: "663b7e212bdcc11befd50c7e",
        hip: 20889,
        ra: 4.47694409,
        dec: 19.18043252,
        dist: 44.712,
        x0: 16.396,
        y0: 38.917,
        z0: 14.69,
      },
      {
        _id: "663b7e212bdcc11befd50a34",
        tyc: "1269-1246-1",
        ra: 4.42482999,
        dec: 17.92790534,
        dist: 47.4237,
        x0: 18.084,
        y0: 41.339,
        z0: 14.598,
      },

      {
        _id: "663b7e212bdcc11befd50868",
        hip: 20455,
        ra: 4.38224798,
        dec: 17.54251527,
        dist: 49.2388,
        x0: 19.295,
        y0: 42.801,
        z0: 14.841,
      },
      {
        _id: "663b7e202bdcc11befd4f136",
        hip: 17847,
        ra: 3.81937298,
        dec: 24.05341343,
        dist: 123.1762,
        x0: 60.782,
        y0: 94.643,
        z0: 50.205,
      },
    ],
    Cancer: [
      {
        _id: "663b7e2f2bdcc11befd616fe",
        hip: 40526,
        ra: 8.27525572,
        dec: 9.1855446,
        dist: 98.9845,
        x0: -54.824,
        y0: 80.886,
        z0: 15.801,
      },
      {
        _id: "663b7e302bdcc11befd63726",
        hip: 42911,
        ra: 8.74475018,
        dec: 18.15430914,
        dist: 40.032,
        x0: -25.042,
        y0: 28.634,
        z0: 12.473,
      },
      {
        _id: "663b7e302bdcc11befd63726",
        hip: 42911,
        ra: 8.74475018,
        dec: 18.15430914,
        dist: 40.032,
        x0: -25.042,
        y0: 28.634,
        z0: 12.473,
      },
      {
        _id: "663b7e302bdcc11befd635ae",
        hip: 42806,
        ra: 8.72142952,
        dec: 21.46849861,
        dist: 53.6392,
        x0: -32.631,
        y0: 37.775,
        z0: 19.631,
      },
      {
        _id: "663b7e302bdcc11befd6395e",
        hip: 43103,
        ra: 8.77828306,
        dec: 28.75990042,
        dist: 101.5228,
        x0: -59.175,
        y0: 66.477,
        z0: 48.847,
      },
      {
        _id: "663b7e302bdcc11befd635ae",
        hip: 42806,
        ra: 8.72142952,
        dec: 21.46849861,
        dist: 53.6392,
        x0: -32.631,
        y0: 37.775,
        z0: 19.631,
      },
      {
        _id: "663b7e2f2bdcc11befd61b46",
        hip: 40843,
        ra: 8.33440576,
        dec: 27.21770692,
        dist: 18.2233,
        x0: -9.299,
        y0: 13.272,
        z0: 8.335,
      },

      {
        _id: "663b7e302bdcc11befd6395e",
        hip: 43103,
        ra: 8.77828306,
        dec: 28.75990042,
        dist: 101.5228,
        x0: -59.175,
        y0: 66.477,
        z0: 48.847,
      },
      {
        _id: "663b7e312bdcc11befd64555",
        hip: 44066,
        ra: 8.9747827,
        dec: 11.85768694,
        dist: 57.7367,
        x0: -39.69,
        y0: 40.218,
        z0: 11.864,
      },
    ],
    virgo: [
      {
        _id: "663b7e3c2bdcc11befd74ae6",
        hip: 71957,
        ra: 14.717673,
        dec: -5.658207,
        dist: 18.2715,
        x0: -13.771,
        y0: -11.872,
        z0: -1.801,
      },
      {
        _id: "663b7e3c2bdcc11befd737a9",
        hip: 69701,
        ra: 14.266908,
        dec: -6.000547,
        dist: 22.237,
        x0: -18.334,
        y0: -12.368,
        z0: -2.325,
      },
      {
        _id: "663b7e3b2bdcc11befd73560",
        hip: 69427,
        ra: 14.21492928,
        dec: -10.2737004,
        dist: 86.2979,
        x0: -71.034,
        y0: -46.526,
        z0: -15.391,
      },
      {
        _id: "663b7e3a2bdcc11befd713bc",
        hip: 65474,
        ra: 13.41989015,
        dec: -11.16124491,
        dist: 76.5697,
        x0: -69.991,
        y0: -27.286,
        z0: -14.822,
      },
      {
        _id: "663b7e3a2bdcc11befd71a54",
        hip: 66249,
        ra: 13.57822,
        dec: -0.59582,
        dist: 22.7118,
        x0: -20.799,
        y0: -9.119,
        z0: -0.236,
      },
      {
        _id: "663b7e3b2bdcc11befd72d84",
        hip: 68520,
        ra: 14.02744238,
        dec: 1.54453325,
        dist: 69.5262,
        x0: -59.938,
        y0: -35.182,
        z0: 1.874,
      },
      {
        _id: "663b7e3c2bdcc11befd74d43",
        hip: 72220,
        ra: 14.77081222,
        dec: 1.89288176,
        dist: 41.1838,
        x0: -30.798,
        y0: -27.308,
        z0: 1.36,
      },
      {
        _id: "663b7e3b2bdcc11befd72d84",
        hip: 68520,
        ra: 14.02744238,
        dec: 1.54453325,
        dist: 69.5262,
        x0: -59.938,
        y0: -35.182,
        z0: 1.874,
      },
      {
        _id: "663b7e3a2bdcc11befd71a54",
        hip: 66249,
        ra: 13.57822,
        dec: -0.59582,
        dist: 22.7118,
        x0: -20.799,
        y0: -9.119,
        z0: -0.236,
      },
      {
        _id: "663b7e392bdcc11befd6ff74",
        hip: 63090,
        ra: 12.92672454,
        dec: 3.39747144,
        dist: 60.8273,
        x0: -58.942,
        y0: -14.588,
        z0: 3.605,
      },
      {
        _id: "663b7e392bdcc11befd703cd",
        hip: 63608,
        ra: 13.03627731,
        dec: 10.95914863,
        dist: 33.1006,
        x0: -31.308,
        y0: -8.709,
        z0: 6.293,
      },
      {
        _id: "663b7e392bdcc11befd6ff74",
        hip: 63090,
        ra: 12.92672454,
        dec: 3.39747144,
        dist: 60.8273,
        x0: -58.942,
        y0: -14.588,
        z0: 3.605,
      },
      {
        _id: "663b7e392bdcc11befd6f621",
        hip: 61941,
        ra: 12.694345,
        dec: -1.449375,
        dist: 11.685,
        x0: -11.489,
        y0: -2.112,
        z0: -0.296,
      },
      {
        _id: "663b7e3a2bdcc11befd713bc",
        hip: 65474,
        ra: 13.41989015,
        dec: -11.16124491,
        dist: 76.5697,
        x0: -69.991,
        y0: -27.286,
        z0: -14.822,
      },
      {
        _id: "663b7e392bdcc11befd6f621",
        hip: 61941,
        ra: 12.694345,
        dec: -1.449375,
        dist: 11.685,
        x0: -11.489,
        y0: -2.112,
        z0: -0.296,
      },
      {
        _id: "663b7e382bdcc11befd6e6c9",
        hip: 60030,
        ra: 12.31119956,
        dec: -0.78718713,
        dist: 114.9812,
        x0: -114.589,
        y0: -9.356,
        z0: -1.58,
      },
      {
        _id: "663b7e372bdcc11befd6d03e",
        hip: 57380,
        ra: 11.76432288,
        dec: 6.52938127,
        dist: 101.5783,
        x0: -100.727,
        y0: 6.223,
        z0: 11.551,
      },
    ],
    libra: [
      {
        _id: "663b7e3f2bdcc11befd781cb",
        hip: 77853,
        ra: 15.89709401,
        dec: -16.72929324,
        dist: 51.6529,
        x0: -25.878,
        y0: -42.158,
        z0: -14.868,
      },
      {
        _id: "663b7e3e2bdcc11befd77344",
        hip: 76333,
        ra: 15.592105,
        dec: -14.789537,
        dist: 49.3601,
        x0: -28.132,
        y0: -38.552,
        z0: -12.6,
      },
      {
        _id: "663b7e3d2bdcc11befd7648b",
        hip: 74785,
        ra: 15.283449,
        dec: -9.382917,
        dist: 56.7537,
        x0: -36.549,
        y0: -42.42,
        z0: -9.253,
      },

      {
        _id: "663b7e3d2bdcc11befd750ab",
        hip: 72622,
        ra: 14.84797594,
        dec: -16.04177696,
        dist: 23.2396,
        x0: -16.409,
        y0: -15.152,
        z0: -6.422,
      },
      {
        _id: "663b7e3d2bdcc11befd75a8b",
        hip: 73714,
        ra: 15.06783762,
        dec: -25.28196292,
        dist: 79.7534,
        x0: -50.079,
        y0: -51.89,
        z0: -34.061,
      },
      {
        _id: "663b7e3e2bdcc11befd77344",
        hip: 76333,
        ra: 15.592105,
        dec: -14.789537,
        dist: 49.3601,
        x0: -28.132,
        y0: -38.552,
        z0: -12.6,
      },
    ],
    SagittariusSchütze: [
      {
        _id: "663b7e492bdcc11befd8572b",
        hip: 95294,
        ra: 19.38698243,
        dec: -44.79978637,
        dist: 42.806,
        x0: 10.788,
        y0: -28.394,
        z0: -30.162,
      },
      {
        _id: "663b7e4b2bdcc11befd87e30",
        hip: 98032,
        ra: 19.921027,
        dec: -41.86827352,
        dist: 55.7414,
        x0: 20.007,
        y0: -36.37,
        z0: -37.203,
      },
      {
        _id: "663b7e492bdcc11befd857e7",
        hip: 95347,
        ra: 19.39810471,
        dec: -40.61593839,
        dist: 55.2218,
        x0: 15.003,
        y0: -39.142,
        z0: -35.949,
      },
      {
        _id: "663b7e4b2bdcc11befd87e30",
        hip: 98032,
        ra: 19.921027,
        dec: -41.86827352,
        dist: 55.7414,
        x0: 20.007,
        y0: -36.37,
        z0: -37.203,
      },
      {
        _id: "663b7e4b2bdcc11befd883a8",
        hip: 98412,
        ra: 19.99560529,
        dec: -35.27630705,
        dist: 205.6676,
        x0: 83.784,
        y0: -145.504,
        z0: -118.777,
      },
      {
        _id: "663b7e4b2bdcc11befd8877f",
        hip: 98688,
        ra: 20.04429991,
        dec: -27.70984656,
        dist: 137.5942,
        x0: 62.126,
        y0: -104.781,
        z0: -63.981,
      },
      {
        _id: "663b7e492bdcc11befd8667f",
        hip: 96406,
        ra: 19.60045931,
        dec: -24.71908406,
        dist: 81.1725,
        x0: 29.999,
        y0: -67.356,
        z0: -33.944,
      },
      {
        _id: "663b7e482bdcc11befd843bd",
        hip: 93864,
        ra: 19.11566829,
        dec: -27.67042392,
        dist: 37.2856,
        x0: 9.508,
        y0: -31.623,
        z0: -17.315,
      },
      {
        _id: "663b7e472bdcc11befd83606",
        hip: 92855,
        ra: 18.92108797,
        dec: -26.29659428,
        dist: 69.8324,
        x0: 14.951,
        y0: -60.794,
        z0: -30.937,
      },
      {
        _id: "663b7e482bdcc11befd838ef",
        hip: 93085,
        ra: 18.962167,
        dec: -21.10665345,
        dist: 118.917,
        x0: 27.65,
        y0: -107.438,
        z0: -42.823,
      },
      {
        _id: "663b7e482bdcc11befd84128",
        hip: 93683,
        ra: 19.07805048,
        dec: -21.7414935,
        dist: 41.9955,
        x0: 10.864,
        y0: -37.465,
        z0: -15.556,
      },
      {
        _id: "663b7e492bdcc11befd85097",
        hip: 94820,
        ra: 19.29391079,
        dec: -18.95291023,
        dist: 147.5524,
        x0: 46.374,
        y0: -131.622,
        z0: -47.924,
      },
      {
        _id: "663b7e492bdcc11befd85578",
        hip: 95168,
        ra: 19.36121085,
        dec: -17.84720025,
        dist: 39.9982,
        x0: 13.283,
        y0: -35.681,
        z0: -12.259,
      },
      {
        _id: "663b7e492bdcc11befd85097",
        hip: 94820,
        ra: 19.29391079,
        dec: -18.95291023,
        dist: 147.5524,
        x0: 46.374,
        y0: -131.622,
        z0: -47.924,
      },
      {
        _id: "663b7e482bdcc11befd84128",
        hip: 93683,
        ra: 19.07805048,
        dec: -21.7414935,
        dist: 41.9955,
        x0: 10.864,
        y0: -37.465,
        z0: -15.556,
      },
      {
        _id: "663b7e482bdcc11befd838ef",
        hip: 93085,
        ra: 18.962167,
        dec: -21.10665345,
        dist: 118.917,
        x0: 27.65,
        y0: -107.438,
        z0: -42.823,
      },
      {
        _id: "663b7e472bdcc11befd83606",
        hip: 92855,
        ra: 18.92108797,
        dec: -26.29659428,
        dist: 69.8324,
        x0: 14.951,
        y0: -60.794,
        z0: -30.937,
      },
      {
        _id: "663b7e472bdcc11befd82ab7",
        hip: 92041,
        ra: 18.76094075,
        dec: -26.99077697,
        dist: 73.3676,
        x0: 12.938,
        y0: -64.084,
        z0: -33.298,
      },
      {
        _id: "663b7e482bdcc11befd83eb2",
        hip: 93506,
        ra: 19.043532,
        dec: -29.880105,
        dist: 27.0416,
        x0: 6.326,
        y0: -22.578,
        z0: -13.472,
      },
      {
        _id: "663b7e482bdcc11befd843bd",
        hip: 93864,
        ra: 19.11566829,
        dec: -27.67042392,
        dist: 37.2856,
        x0: 9.508,
        y0: -31.623,
        z0: -17.315,
      },
      {
        _id: "663b7e472bdcc11befd82ab7",
        hip: 92041,
        ra: 18.76094075,
        dec: -26.99077697,
        dist: 73.3676,
        x0: 12.938,
        y0: -64.084,
        z0: -33.298,
      },
      {
        _id: "663b7e452bdcc11befd8118f",
        hip: 90185,
        ra: 18.40287398,
        dec: -34.3843146,
        dist: 43.9367,
        x0: 3.817,
        y0: -36.058,
        z0: -24.813,
      },
      {
        _id: "663b7e452bdcc11befd809a6",
        hip: 89642,
        ra: 18.29378698,
        dec: -36.76168819,
        dist: 44.7427,
        x0: 2.754,
        y0: -35.739,
        z0: -26.778,
      },
      {
        _id: "663b7e452bdcc11befd8118f",
        hip: 90185,
        ra: 18.40287398,
        dec: -34.3843146,
        dist: 43.9367,
        x0: 3.817,
        y0: -36.058,
        z0: -24.813,
      },
      {
        _id: "663b7e442bdcc11befd7fb5c",
        hip: 88635,
        ra: 18.09680182,
        dec: -30.42409858,
        dist: 29.7,
        x0: 0.649,
        y0: -25.602,
        z0: -15.04,
      },
      {
        _id: "663b7e432bdcc11befd7e698",
        hip: 87072,
        ra: 17.7926735,
        dec: -27.83079164,
        dist: 356.4158,
        x0: -17.099,
        y0: -314.725,
        z0: -166.397,
      },
      {
        _id: "663b7e442bdcc11befd7fb5c",
        hip: 88635,
        ra: 18.09680182,
        dec: -30.42409858,
        dist: 29.7,
        x0: 0.649,
        y0: -25.602,
        z0: -15.04,
      },
      {
        _id: "663b7e452bdcc11befd80dd5",
        hip: 89931,
        ra: 18.34990047,
        dec: -29.8281024,
        dist: 127.441,
        x0: 10.113,
        y0: -110.094,
        z0: -63.389,
      },
      {
        _id: "663b7e452bdcc11befd8118f",
        hip: 90185,
        ra: 18.40287398,
        dec: -34.3843146,
        dist: 43.9367,
        x0: 3.817,
        y0: -36.058,
        z0: -24.813,
      },
      {
        _id: "663b7e452bdcc11befd80dd5",
        hip: 89931,
        ra: 18.34990047,
        dec: -29.8281024,
        dist: 127.441,
        x0: 10.113,
        y0: -110.094,
        z0: -63.389,
      },
      {
        _id: "663b7e472bdcc11befd82ab7",
        hip: 92041,
        ra: 18.76094075,
        dec: -26.99077697,
        dist: 73.3676,
        x0: 12.938,
        y0: -64.084,
        z0: -33.298,
      },

      {
        _id: "663b7e462bdcc11befd81624",
        hip: 90496,
        ra: 18.46617795,
        dec: -25.42170006,
        dist: 23.2993,
        x0: 2.562,
        y0: -20.887,
        z0: -10.002,
      },

      {
        _id: "663b7e452bdcc11befd804f4",
        hip: 89341,
        ra: 18.229392,
        dec: -21.058834,
      },
      {
        _id: "663b7e462bdcc11befd81624",
        hip: 90496,
        ra: 18.46617795,
        dec: -25.42170006,
        dist: 23.2993,
        x0: 2.562,
        y0: -20.887,
        z0: -10.002,
      },
      {
        _id: "663b7e452bdcc11befd80dd5",
        hip: 89931,
        ra: 18.34990047,
        dec: -29.8281024,
        dist: 127.441,
        x0: 10.113,
        y0: -110.094,
        z0: -63.389,
      },
    ],
  };

  let selectedArray;
  $: if (selectedArray) {
    updateLines(selectedArray);
  }

  function updateLines(arrayName) {
    // lineGroup.clear(); // Löscht nur die Linien in der Gruppe
    console.log(arrayName);
    let array = arrays[arrayName];
    if (array) {
      for (let i = 1; i < array.length - 1; i++) {
        addLine(array[i], array[i + 1], array[0]);
      }
    } else {
      console.error("Unbekanntes Array:", arrayName);
    }
  }

  function addLine(start, end, center) {
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
    lineGroup.add(line); // Füge die Linie zur Gruppe hinzu
    scene.add(lineGroup);
    // console.log(center);
    moveToConstellation(new THREE.Vector3(center.y, center.z, center.x));
  }

  function moveToConstellation(newTarget) {
    // Aktuelle Position der Kamera speichern

    // Setze das neue Ziel für die OrbitControls
    controls.target.copy(newTarget);

    // Setze die Kameraposition zurück, um sicherzustellen, dass die Kamera nicht bewegt wird
    camera.position.set(0, 0, 0.00005);

    // Notwendig, um die Änderungen zu verarbeiten
    controls.update();
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

  let translateX = "-100%"; // Zustand der X-Translation des Containers

  function toggleContainer() {
    translateX = translateX === "-100%" ? "-73%" : "-100%"; // Schaltet zwischen den Positionen um
  }


  let infoContent = ""; // Inhalt für das info-overlay
  let displayInfoOverlay = "none"; // Steuert die Anzeige des info-overlays
  let showInfoOverlay = false;


  function showInfo(element) {
    infoContent = element;
    selectedArray = element;
    showInfoOverlay = true;
    displayInfoOverlay = "block"; // Zeigt das info-overlay an
    console.log("show info");
  }

  function hideInfo() {
    displayInfoOverlay = "none"; // Verbirgt das info-overlay
    showInfoOverlay = false;
  }
  import svgURL from "./assets/constel.svg";
  import sunURL from "./assets/sun.svg";
  import eyeURL from './assets/eye.svg';
</script>

<main>
  <div
    id="info-overlay"
    class="overlay"
    style="--displayInfoOverlay: {displayInfoOverlay};"
  >
  {infoContent}
  <p>Stern 1</p>
  <p>Stern 2</p>
  <p>Stern 3</p>
  <p>Stern 4</p>
  </div>
  <div id="container" style="--translateX: {translateX};">
    <div id="selection-overlay" class="overlay2">
      <button class='constbtn' on:click={() => showInfo("Wassermann")}><img src={svgURL}></button>
      <button class='constbtn' on:click={() => showInfo("Widder")}><img src={svgURL}></button>
    </div>
    <div id="options">
      <button class='constbtn' on:click={toggleContainer}><img src={svgURL}></button>
      <button class='constbtn' on:click={returnToSun}><img src={sunURL}></button>
      <button class='constbtn'><img src={eyeURL}></button>
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
  .constbtn {
    padding: 5px;
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

  #options {
    padding-left: 10px;
    padding-right: 10px;
    background-color: rgba(255, 255, 255, 0.1);
    color: white;
    max-height: 40px;
    border-bottom-right-radius: 10px;
    margin:0;
    display: flex;
    gap: 10px;
  }

  #info-overlay {
    position: absolute;
    display: var(--displayInfoOverlay, none);
    right: 10px;
    top: 10px;
  }
</style>
