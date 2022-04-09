---
layout: docs
title: Sequential Monte Carlo
permalink: docs/casestudies/smc
---

# Sequential Monte Carlo


```effekt:prelude:hide
import immutable/list
import immutable/option
import mutable/array
import unsafe/cont
```

We define the following effect to model probabilistic processes.
```
effect SMC {
  def resample(): Unit
  def uniform(): Double
  def score(d: Double): Unit
}

effect Measure[R] {
  def measure(weight: Double, data: R): Unit
}
```

We can use the effect to define some probabilistic programs.
```
def bernoulli(p: Double) = uniform() > p

def geometric(p: Double): Int / SMC = {
  resample();
  val x = bernoulli(p);
  if (x) {
    score(log(1.5));
    1 + geometric(p)
  } else { 1 }
}
```


## A SMC Handler
A particle consists of its current _weight_ (or "score"), the _age_ (number of resampling generations it survived -- not used at the moment),
and the continuation that corresponds to the remainder of its computation.
```
record Particle(weight: Double, age: Int, cont: Cont[Unit, Unit])
```

Using the `Particle` data type, we can define our SMC handler as follows:
```
def smc[R](numberOfParticles: Int) {
  // should maintain the number of particles.
  resample: List[Particle] => List[Particle]
} { p: () => R / SMC } = {
  var weight = 1.0;
  var particles: List[Particle] = Nil()
  var age = 0;

  def checkpoint(cont: Cont[Unit, Unit]) =
    particles = Cons(Particle(weight, age, cont), particles)

  def run(p: Particle): Unit = {
    weight = p.weight;
    age = p.age;
    p.cont.apply(())
  }

  def run() = {
    val ps = resample(particles);
    particles = Nil();
    ps.foreach { p => p.run }
  }

  repeat(numberOfParticles) {
    weight = 1.0;
    try { val res = p(); measure(weight, res) }
    with SMC {
      def resample() = checkpoint(cont { t => resume(t) })
      def uniform() = resume(random())
      def score(d) = { weight = weight * d; resume(()) }
    }
  }

  while (not(particles.isEmpty)) { run() }
}
```
It runs `numberOfParticles`-many instances of the provided program `p` under a handler that collects the
continuation whenever a particle encounters a call to `resample`. Once all particles either finished their
computation or hit a `resample`, the handler passes the list of live particles to the argument function `resample`.
This argument function then can reorder and redistribute the particles.

## Resampling
We now implement such a (naive) resampling function. It proceeds by first filling an array (100 times the particle count)
with a number copies of the particles relative to their weight.
Then it picks new particles at random.
```
def resampleUniform(ps: List[Particle]): List[Particle] = {
  val total = ps.totalWeight
  val numberOfParticles = ps.size
  val targetSize = numberOfParticles * 100;
  var buckets = emptyArray[Particle](targetSize);
  var bucketSize = 0;

  def add(p: Particle) = {
    buckets.put(bucketSize, p);
    bucketSize = bucketSize + 1
  }

  // fill buckets
  ps.foreach { p =>
    val prob = p.weight / total
    val numberOfBuckets = (prob * targetSize.toDouble).toInt;
    repeat(numberOfBuckets) { add(p) }
  }

  // select new particles by drawing at random
  var newParticles: List[Particle] = Nil();
  repeat(numberOfParticles) {
    val index = (random() * (bucketSize - 1).toDouble).toInt
    buckets.get(index).foreach { p =>
      newParticles = Cons(p, newParticles)
    }
  }
  newParticles
}

// helper function to compute the total weight
def totalWeight(ps: List[Particle]): Double = {
  var totalWeight = 0.0
  ps.foreach { case Particle(w, _, _) =>
    totalWeight = totalWeight + w
  }
  totalWeight
}
```



## Importance Sampling
Of course the above handler is not the only one. We can define an even simpler handler
that performs importance sampling by sequentially running each particle to the end.
```
def importance[R](n: Int) { p : R / SMC } =
  n.repeat {
    var currentWeight = 1.0;
    try {
      val result = p();
      measure(currentWeight, result)
    } with SMC {
      def resample() = resume(())
      def uniform() = resume(random())
      def score(d) = { currentWeight = currentWeight * d; resume(()) }
    }
  }
```

### Running the Examples

```effekt:hide
extern def sleep(n: Int): Unit =
  "$effekt.callcc(k => window.setTimeout(() => k(null), n))"

// here we set a time out to allow rerendering
extern def reportMeasurementJS[R](w: Double, d: R): Unit =
  "$effekt.callcc(k => { showPoint(w, d); window.setTimeout(() => k(null), 0)})"

// here we set a time out to allow rerendering
extern pure def setupGraphJS(): Unit =
  "setup()"
```
To visualize the results, we define the following helper function `report` that
handlers `Measure` effects by adding the data points to a graph (below).
```
def report[R](interval: Int) { prog: Unit / Measure[R] } =
  try { setupGraphJS(); prog() }
  with Measure[R] {
    def measure(w, d) = {
      reportMeasurementJS(w, d);
      sleep(interval);
      resume(())
    }
  }
```

Running SMC and importance sampling now is a matter of composing the handlers.
```
def runSMC(numberOfParticles: Int) =
  report[Int](20) {
    smc(numberOfParticles) { ps => resampleUniform(ps) } { geometric(0.5) }
  }
```

```
def runImportance(numberOfParticles: Int) =
  report[Int](20) {
    importance(numberOfParticles) { geometric(0.5) }
  }
```


In the below REPL you can try the examples. Click `run` and then try entering `runSMC(100)` (then click `run` again):
```effekt:repl
runSMC(100)
```


<div id="graph"></div>
<p>Particles: <span id="count"></span></p>

<script src="https://d3js.org/d3.v6.js"></script>


<script>

// Good articles on d3:
// - https://bost.ocks.org/mike/selection/
// - https://bost.ocks.org/mike/join/

var count = 0
const counter = document.getElementById("count")


// holds our data
var data = []

const margin = {top: 30, right: 30, bottom: 70, left: 60},
    width = 460 - margin.left - margin.right,
    height = 400 - margin.top - margin.bottom;

// append the svg object to the body of the page
const svg = d3.select("#graph")
  .append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)
  .append("g")
    .attr("transform",
          "translate(" + margin.left + "," + margin.top + ")");

// X axis
const x = d3.scaleBand().range([ 0, width ]).padding(0.2);

// Y axis
const y = d3.scaleLinear().domain([0, 1]).range([ height, 0]);

// Axis
const xaxis = svg.append("g")
  .attr("class", "x axis")
  .attr("transform", "translate(0," + height + ")")
  .call(d3.axisBottom(x))

const yaxis = svg.append("g")
  .attr("class", "y axis")
  .call(d3.axisLeft(y));

const bar = svg.selectAll(".bar")
  .data(data)


function setup() {
  data = []
  count = 0
  counter.innerHTML = ""
}

// returns an array from n to m (including both ends)
function range(n, m) {
  var arr = [];
  while (n <= m) { arr.push(n); n++ }
  return arr
}

function render(data) {
  const minValue = d3.min(data, function(d) { return d.value; })
  const maxValue = d3.max(data, function(d) { return d.value; })

  // this isn't yet fully working...

  // use this for non-linear data: data.map(function(d) { return d.value; })
  x.domain(range(minValue, maxValue))
  xaxis.call(d3.axisBottom(x))

  // use this for rescaling the y axis:
  //y.domain([0, d3.max(data, function(d) { return d.weight; })]);
  //yaxis.call(d3.axisLeft(y));

  function xValue(d) { return x(d.value) }
  function yWeight(d) { return y(d.weight) }
  function barHeight(d) { return height - y(d.weight) }
  const barWidth = x.bandwidth()


  // Bars
  const bar = svg.selectAll(".bar")
    .data(data);

  bar.exit()
    .remove();

  bar.enter()
    // code that will be run to add new elements
    .append("g")
      .attr("class", "bar")
    .append("rect")
      .attr("x", xValue)
      .attr("y", yWeight)
      .attr("width", barWidth)
      .attr("height", barHeight)
      .attr("fill", "#69b3a2")
    .merge(bar)
      // code that will be run to update elements
      .transition()
      .duration(0)
      .select("rect")
        .attr("x", xValue)
        .attr("y", yWeight)
        .attr("width", barWidth)
        .attr("height", barHeight)
}


function normalize(data) {
  var sum = 0;
  for (let entry of data) {
    sum = sum + entry.weight
  }
  var newdata = []
  for (let entry of data) {
     newdata.push({ value: entry.value, weight: entry.weight / sum })
  }
  return newdata;
}

function showPoint(weight, value) {
  count = count + 1;
  counter.innerHTML = count;

  var found = false;
  for (var i = 0; i < data.length; i++) {
    let el = data[i]
    if (el.value === value) {
      el.weight = el.weight + weight;
      found = true
    }
  }
  if (!found) {
    data.push({ weight, value })
  }
  render(normalize(data))
}

</script>