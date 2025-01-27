---
title: Tworzenie tła z poruszającymi się cząsteczkami
date: 2017-03-17
categories: [Programming]
permalink: /pl/2017/03/tworzenie-tla-z-poruszajacymi-sie-czasteczkami/
redirect_from:
  - /2017/03/tworzenie-tla-z-poruszajacymi-sie-czasteczkami/
image:
  path: /assets/img/particles/img.png
---
W tym wpisie przedstawię w jaki sposób stworzyć tło z poruszającymi się cząsteczkami. Całość oparta będzie o element `<canvas>` oraz ES6. Wykorzystam tutaj kod, który już kiedyś stworzyłem jako część gry Paper Soccer.

## Cel
Celem jest stworzenie tła wykorzystanego tutaj Paper Soccer. Oczywiście mam już ten kod gotowy, jednak chciałbym stworzyć coś na kształt biblioteki, która pozwoli w łatwy sposób podpiąć się pod dany element `<canvas>` i utworzyć w nim takie tło oraz umożliwić konfigurację parametrów takich jak:
- kolor
- wielkość cząsteczek
- ilość cząsteczek
- grubość linii
- etc.

## Lecimy z tematem
Biblioteka nie będzie wymagała żadnych zależności, jedynie do stworzenia produkcyjnej wersji wykorzystany zostanie webpack oraz  babel do kompiliacji ES6 do ES5. Struktura katalogów prezentuje się następująco:

- /demo – przykład wykorzystania
- /dist – skompilowana  do ES5 biblioteka oraz zminifikowana
- /src – pliki zawierające poszczególne klasy
- /test – testy

Najbardziej interesuje nas tutaj katalog /src, w którym zawarty będzie kod podzielony na poszczególne moduły(klasy):

- Particle – klasa reprezentująca pojedynczą cząsteczkę
- ParticleCreator – klasa mająca za zadanie tworzenie cząsteczek o określonych parametrach
- Drawer – klasa odpowiedzialna za rysowanie linii oraz cząsteczek
- Background – klasa odpowiedzialna za całe sterowanie, tworząca obiekty klas ParticleCreator oraz Drawer

Standard ES6 według mnie usprawnił tworzenie klas, przynajmniej teraz to wygląda bardziej ludzko oraz w podobnej do innych języków składni.

## Reprezentacja pojedynczej cząsteczki
```javascript
class Particle {
  /**
   * Particle constructor
   *
   * @param int x
   * @param int y
   * @param array vel
   * @param int radius
   */
  constructor(x, y, vel, radius) {
    this.x = x;
    this.y = y;
    this.vel = vel;
    this.radius = radius;
  }
  
  /**
   * Move particle with set velocity
   *
   * @param int maxX
   * @param int maxY
   */
  move(maxX, maxY) {
    this.x += this.vel.x;
    this.y += this.vel.y;
    
    if (this.isOffXEdge(maxX)) {
      this.vel.x = -this.vel.x;
    }
    
    if (this.isOffYEdge(maxY)) {
      this.vel.y = -this.vel.y;
    }
  }
  
  /**
   * Check that particle is off the X edge.
   *
   * @param int maxX
   * @return bool
   */
  isOffXEdge(maxX) {
    return this.x > maxX || this.x < 0;
  }
  
  /**
   * Check that particle is off the Y edge.
   *
   * @param maxY
   * @return bool
   */
  isOffYEdge(maxY) {
    return this.y > maxY || this.y < 0
  }
}
export default Particle;
```

Powyższy kod przedstawia klasę reprezentującą pojedynczą cząsteczkę. Mamy tutaj konstruktor, który ustawia kilka wartości: współrzędne, prędkość oraz promień. Metoda move() odpowiedzialna jest za wykonanie ruchu cząsteczki, czyli zwiększenie współrzędnych o odpowiednie wartości określone przez atrybut vel. Wywoływana ona będzie w klasie Background, podczas odświeżania animacji, co w efekcie powodowało będzie ruch cząsteczki. Prędkość jest tutaj przedstawiana za pomocą obiektu z dwiema wartościami x oraz y, o które podczas wywołania metody move() zwiększane są odpowiednio współrzędne cząsteczki. W przypadku, gdy cząsteczka wyjdzie poza daną krawędź ekranu zmienia zwrot na przeciwny i porusza się  w drugą stronę, co tworzy wrażenie odbijania się cząsteczki od krawędzi. Warunki tego wyjścia poza ekran sprawdzane są w dwóch instrukcjach warunkowych dla każdej z osi.

## Tworzenie cząsteczek

```javascript
import Particle from './Particle';

class ParticleCreator {
  /**
   * ParticleCreator constructor.
   *
   * @param object settings
   * @param object canvas
   */
  constructor(settings, canvas) {
    this.settings = settings;
    this.canvas = canvas;
  }
  
  /**
   * Create many particles with specified count.
   *
   * @param int count
   * @return Particle[]
   */
  createMany(count) {
    let particles = [];
    for (let i = 0; i < count; i++) {
      particles.push(this.create());
    }
    return particles;
  }
  
  /**
   * Create single particle.
   *
   * @return Particle
   */
  create() {
    return new Particle(
      Math.random() * this.canvas.width,
      Math.random() * this.canvas.height, {
        x: (Math.random() - 0.5) * 5, // [-2.5, 2.5)
        y: (Math.random() - 0.5) * 5 // [-2.5, 2.5)
      },
      this.settings.particleRadius
    );
  }
}
export default ParticleCreator;
```

Powyższa klasa odpowiedzialna jest za utworzenie cząsteczek o określonych parametrach, wykorzystywana jest tutaj klasa omówiona wcześniej – Particle. Konstruktor przyjmuje tutaj dwa parametry: settings oraz canvas. Pierwszy z nich to tablica z ustawieniami biblioteki, która zawiera dane określające ile cząsteczek stworzyć oraz o jakim promieniu. Drugi parametr(canvas) to po prostu element canvas, który tutaj wykorzystywany jest do pobrania maksymalnej szerokości oraz wysokości. Jak już wcześniej napisałem promień pobierany jest z ustawień biblioteki, natomiast współrzędne oraz prędkość losowane są z odpowiednich przedziałów. Math.random() zwraca losową wartość z przedziału [0, 1), więc przedziały dla współrzędnych będą następujące:
- x – [0, szerokość okna)
- y  – [0, wysokość okna)
Przedziały dla prędkości zostały zapisane w komentarzu obok.

## Rysowanie
```javascript
class Drawer {
  /**
   * Drawer constructor
   *
   * @param object settings
   * @param object ctx
   */
  constructor(settings, ctx) {
    this.settings = settings;
    this.ctx = ctx;
  }
  
  /**
   * Draw lines from this particle to other particles where distance between them is lower than specified distance
   *
   * @param Particle[] particles
   * @param Particle particle
   */
  drawLines(particles, particle) {
    this.ctx.beginPath();
    particles.forEach(secondParticle => {
      const dist = Math.hypot(particle.x - secondParticle.x, particle.y - secondParticle.y);
      if (dist < this.settings.particleDistance) {
        this.drawLine(particle, secondParticle);
      }
    });
    this.ctx.stroke();
  }
  
  /**
   * Draw line between two particles
   *
   * @param firstParticle
   * @param secondParticle
   */
  drawLine(firstParticle, secondParticle) {
    this.ctx.strokeStyle = this.settings.lineColor;
    this.ctx.lineWidth = this.settings.lineWidth;
    this.ctx.moveTo(firstParticle.x, firstParticle.y);
    this.ctx.lineTo(secondParticle.x, secondParticle.y);
  }
  
  /**
   * Draw Particle
   *
   * @param Particle particle
   */
  drawParticle(particle) {
    this.ctx.beginPath();
    this.ctx.arc(particle.x, particle.y, particle.radius, 0, Math.PI * 2);
    this.ctx.fillStyle = this.settings.particleColor;
    this.ctx.fill();
  }
}
export default Drawer;
```

W powyższym kodzie zawarta została klasa odpowiedzialna za rysowanie cząsteczek oraz linii między nimi. Konstruktor przyjmuje tutaj również dwa parametry. Settings zostało już wspomniane, są to po prostu ustawienia biblioteki. Natomiast ctx to:
```javascript
this.ctx = this.canvas.getContext('2d');
```

czyli kontekst umożliwiający rysowanie.

> The HTMLCanvasElement.getContext() method returns a drawing context on the canvas
> https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/getContext

Metody drawLine() oraz drawParticle() odpowiedzialne są po prostu za rysowanie odpowiednio linii oraz koła. Po wyjaśnienie instrukcji tam zawartych odsyłam do dokumentacji, jest to po prostu zwykłe API służące do rysowania tych elementów.

Linie pomiędzy cząsteczkami rysowane są w taki sposób, że przy każdym odświeżeniu animacji dla każdej cząsteczki sprawdzana jest odległość jej samej z pozostałymi cząsteczkami, jeśli jest ona mniejsza niż wartość określona przez właściwość particleDistance obiektu settings to rysowana jest linia pomiędzy tymi punktami. Metoda Math.hypot() służy do obliczenia pierwiastka kwadratowego sumy kwadratów argumentów. Dzięki niej łatwo oblicza się odległość między punktami.

Metoda drawLines() odpowiada za wysokie zużycie procesora, gdyż jeśli animacja odświeżana jest 60 razy na sekundę, a ta metoda jest wywoływana przy każdym odświeżeniu, to przy założeniu, że mamy 100 cząsteczek na sekundę wykonywane jest wtedy 60*100*100 = 600 000 obliczeń.

## Sterowanie
```javascript
import ParticleCreator from './ParticleCreator';
import Drawer from './Drawer';

class Background {
  /**
   * Background constructor, get canvas element, set options
   *
   * @param string canvasID
   * @param object settings
   */
  constructor(canvasID, settings) {
    this.setSettings(settings);
    this.canvas = document.getElementById(canvasID);
    this.ctx = this.canvas.getContext('2d');
    this.particleCreator = new ParticleCreator(this.settings, this.canvas);
    this.drawer = new Drawer(this.settings, this.ctx);
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
    this.particles = [];
    this.init();
  }
  
  /**
   * Set settings
   *
   * @param object settings
   */
  setSettings(settings) {
    this.settings = {
      particleColor: this.getOption(settings, 'particleColor', '#000000'),
      particleRadius: this.getOption(settings, 'particleRadius', 3),
      particleDistance: this.getOption(settings, 'particleDistance', 100),
      particlesCount: this.getOption(settings, 'particlesCount', 300),
      lineColor: this.getOption(settings, 'lineColor', '#000000'),
      lineWidth: this.getOption(settings, 'lineWidth', 1)
    };
  }
  
  /**
   * Get option
   *
   * @param object settings
   * @param string name
   * @param mixed default
   */
  getOption(settings, name, defaultValue) {
    return settings.hasOwnProperty(name) ? settings[name] : defaultValue;
  }
  
  /**
   * Create particles and start loop
   */
  init() {
    this.particles = this.particleCreator.createMany(this.settings.particlesCount);
    this.loop();
  }
  
  /**
   * Loop, clear and drawParticles
   */
  loop() {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    this.particles.forEach(particle => {
      particle.move(this.canvas.width, this.canvas.height);
      this.drawer.drawLines(this.particles, particle);
      this.drawer.drawParticle(particle);
    });
    window.requestAnimationFrame(this.loop.bind(this));
  }
}
export default Background;
```

Powyżej przedstawiona została ostatnia klasa, która jest klasą główną. Konstruktor przyjmuje tutaj dwa parametry: ID elementu canvas oraz tablicę z ustawieniami. Klasa zawiera w sobie ustawienie właściwości, jeśli nie są one przekazywane z zewnątrz to w metodzie setSettings() ustawiane są wartości domyślne. Kolejno w konstruktorze mamy:
- pobranie elementu canvas na podstawie ID
- pobranie kontekstu
- utworzenie obiektu klasy ParticleCreator do tworzenia cząsteczek
- utworzenie obiektu klasy Drawer do rysowanie linii oraz cząsteczek
- ustawienie wymiarów elementu canvas – fullscreen
- inicjalizacja tablicy przechwowującej cząsteczki
- wywołanie metody init

Metoda init() za pomocą obiektu particleCreator tworzy cząsteczki, następnie wywołuje metodę loop(), w której to:
- czyszczony jest cały ekran
- rysowane są cząsteczki, linie oraz wykonywany jest ruch (zmiana współrzędnych o określoną wartość)

Następnie metoda loop() przekazywana jest jako argument do Window.requestAnimationFrame(), która wykonuje metodę loop() przed odświeżeniem animacji.

Dodatkowo w katalogu /src jest jeszcze plik:
```javascript
import Background from './modules/Background';
export function create(canvasID, settings) {
  return new Background(canvasID, settings);
};
```

Eksportowana jest tutaj funkcja, która ma za zadanie przyjąć parametry oraz utworzyć obiekt klasy Background. Dzięki temu w poniższy prosty sposób możemy skorzystać z tej biblioteki:
```javascript
EasyParticlesBackground.create('background', {
  particleColor: 'rgba(236, 208, 120, 1)',
  particleRadius: 3,
  particleDistance: 100,
  particlesCount: 200,
  lineColor: 'rgba(236, 208, 120, 1)',
  lineWidth: 1
});
```
Dokładniejszy przykład użycia można znaleźć w linkach poniżej.

## Finalny efekt
- [Github](https://github.com/sarven/easy-particles-background)

## TO DO
Póki co biblioteka jest prototypem napisanym na szybko, który pasowałoby rozbudować o rzeczy takie jak:
- testy
- nasłuchiwanie zdarzenia resize, tak aby dostosować tło do zmienionego rozmiaru okna
- optymalizacja – obecnie zużycie procesora jest trochę zbyt duże przez ogromną ilość wykonywanych obliczeń, trzeba byłoby pomyśleć jak to lepiej rozwiązać
- większa ilość opcji konfiguracyjnych

Możliwe, że w przyszłości wrócę do tego tematu o ile będzie jakiekolwiek zainteresowanie, to postaram się to bardziej dopracować.

