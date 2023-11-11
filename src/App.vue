<script setup lang="js">


import {computed, reactive, ref} from "vue";

let sourceCodes = [
  "r=0", // always defect
  "r=1", // always cooperate
  "r = h.length === 0 ? 1 : h[h.length - 1].o;", // tit for tat. round 1 cooperate, then cooperate if last round opp cooperated.
  'r = Math.random() < 0.5;', // random
  "r = (c == s || !c.match('Math')) ? 1 : 0;", // cooperate with self and not using math
  'r = c.length > 15', // cooperate if code is longer than 15 characters
  'r = d === m ? 1 : f(c, d, m, "r=1", c, f, h, i) === 1 && f(c, d, m, "r=0", c, f, h, i) === 0 ? 1 : 0;', // cooperate with self and opponents that simulation says will mirror you
  'r;try{const hC=h.map(()=>({m:1})),hD=h.map(()=>({m:0}));r=d==m||(f(c,d,m,"r=1",c,f,hC)==1&&f(c,d,m,"r=0",c,f,hD)==0)||(d!=m&&h.length&&f(c,d,m,s,c,f,h)==h.slice(-1)[0].o)?1:0}catch(e){r=0}',
  'r = 0; setTimeout(function() { }, 750);',
  'r = c.includes("r = 1") ? 1 : 0;',
  'r = 1; setTimeout(function() { }, 750);',
  'r = h.length === 0 ? 1 : (h[h.length - 1].o === 1 ? h[h.length - 1].m : 1 - h[h.length - 1].m)',
  'let r_ = h.length === 0 ? 1 : (h[h.length - 1].o === 1 ? h[h.length - 1].m : 1 - h[h.length - 1].m); r = Math.random() < 0.95 ? r : 1 - r;',
  'let r_ = h.length === 0 ? 1 : (h[h.length - 1].o === 1 ? h[h.length - 1].m : 1 - h[h.length - 1].m); r = Math.random() < 0.95 ? r_ : 1 - r_;',
  'r = h.every((r) => r.o);',
  'r = 1 && h.every((r) => r.o);',
  "r = 1; r = (d === 2) ? 1 : ((d < 14) ? 0 : h.every((r) => r.o))",

  // examples
  "r = h.length === 0 ? 1 : h[h.length - 1].o; if (h.length === 99) {r = 0};",
  'if (c = "r = h.length === 0 ? 1 : h[h.length - 1].o;") {r=1} else {r = 0};',
  'if (c.includes("eval") || c.includes("f(") || c.includes(";h") || c.includes(" h")) {r = 1} else {r = 0};',

  //mine
  "r = (d === 2) ? 1 : ((d < 14) ? 0 : h.every((r) => r.o))",
];

let programs = sourceCodes.map(sourceCode => {
  return {
    sourceCode,
    scores: [],
  };
});

const interactions = reactive(programs.map(() => {
  return programs.map(() => {
    return []
  });
}));

function getAvgPayoffsForMatchup(payoffHistory) {
  const avg = payoffHistory.reduce((acc, score) => acc + score, 0) / payoffHistory.length
  return avg.toFixed(2)
}

const avgPayoffsForMatchups = computed(() => {
  return interactions.map((row) => {
    return row.map((payoffHistory) => {
      return getAvgPayoffsForMatchup(payoffHistory)
    })
  })
})

const avgPayoffTotalByModel = computed(() => {
  // average columns
  const avgPayoffByModel = []
  for (let i = 0; i < programs.length; i++) {
    let total = 0
    for (let j = 0; j < programs.length; j++) {
      total += parseFloat(avgPayoffsForMatchups.value[j][i])
    }
    total /= programs.length
    avgPayoffByModel.push(total.toFixed(2))
  }

  return avgPayoffByModel
})

function getPayoff(aCooperates, bCooperates) {
  if (aCooperates && bCooperates) {
    return [2, 2];
  } else if (aCooperates && !bCooperates) {
    return [0, 3];
  } else if (!aCooperates && bCooperates) {
    return [3, 0];
  } else {
    return [1, 1];
  }
}

function f(a, d, m, c, s, f, h, i) {
  let startTime = performance.now();

  const func = new Function('d', 'm', 'c', 's', 'f', 'h', 'i',
      `r=9;
       ${a}
       return r;`
  );

  const result = func(d, m, c, s, f, h, i);

  if (performance.now() - startTime > 1000) {
    return "ERR"
  }
  return result
}

function runRound(myProgram, opponentProgram, myId, opponentId, history) {
  const opponentHistory = history.map(({m, o}) => {
    return {
      m: o,
      o: m,
    };
  });

  const myDecision = f(
      myProgram.sourceCode, //a
      opponentId, //d
      myId, // m
      opponentProgram.sourceCode, // c
      myProgram.sourceCode, // s
      f, // f
      history, // h
      opponentHistory, // i
  )

  const opponentDecision = f(
      opponentProgram.sourceCode, //a
      myId, //d
      opponentId, // m
      myProgram.sourceCode, // c
      opponentProgram.sourceCode, // s
      f, // f
      opponentHistory, // h
      history, // i
  )

  if (myDecision === "ERR"){
    console.log('ERR', myProgram, 'against', opponentProgram);
    return
  }
  if (opponentDecision === "ERR"){
    console.log('ERR', opponentProgram, 'against', myProgram);
    return;
  }

  history.push({"m": myDecision, "o": opponentDecision});
}


function runMatch(myProgram, opponentProgram, myId, opponentId) {
  const numMatches = Math.floor(Math.random() * 100+1);
  const history = []

  for (let i = 0; i < numMatches; i++) {
    runRound(myProgram, opponentProgram, myId, opponentId, history);
  }

  const payoffs = history.map(({m, o}) => {
    return getPayoff(m, o);
  });


  interactions[myId][opponentId] = interactions[myId][opponentId].concat(
      payoffs.map(([a, b]) => {
        return a;
      })
  );
  interactions[opponentId][myId] = interactions[opponentId][myId].concat(
      payoffs.map(([a, b]) => {
        return b;
      })
  );

  const myScore = payoffs.reduce((acc, [a, _]) => {
    return acc + a;
  }, 0) / payoffs.length;
  const opponentScore = payoffs.reduce((acc, [_, b]) => {
    return acc + b;
  }, 0) / payoffs.length;

  myProgram.scores.push(myScore);
  opponentProgram.scores.push(opponentScore);
}

const getColor = (avgPayoff) => {
  const hue = (avgPayoff / 3) * 120

  return `hsl(${hue}, 50%, 50%)` //`rgb(${red}, ${green}, 0)`;
};

function runTournament() {
  for (let repeat = 0; repeat < 1; repeat++) {
    for (let i = 0; i < programs.length; i++) {
      for (let j = 0; j < programs.length; j++) {
        runMatch(programs[i], programs[j], i, j);
      }
    }
  }

  for (let i = 0; i < programs.length; i++) {
    programs[i].avgScore = programs[i].scores.reduce((acc, score) => acc + score, 0) / programs[i].scores.length;
  }

  console.log('programs', programs);
}

const simulating = ref(true);

runTournament()

simulating.value = false;

</script>

<template>

  <div v-if="simulating">
    Simulating...
  </div>

  <table style="width: 100%">
    <tr>
      <td v-for="(program, index) in programs" :key="index" class="cell"
      :style="{
        color: 'black',
        backgroundColor: getColor(avgPayoffTotalByModel[index])
      }"
      >
        <p>{{ index }}</p>
        <p> {{avgPayoffTotalByModel[index]}} </p>
      </td>
    </tr>
    <tr v-for="(row, i) in interactions" :key="i">
      <td v-for="(payoffHistory, j) in row" :key="j" class="cell"
      :style="{
        color: 'black',
        backgroundColor: getColor(avgPayoffsForMatchups[i][j])
      }">
        {{ avgPayoffsForMatchups[i][j] }}
      </td>
    </tr>
  </table>



  <div>
    <div v-for="(program, index) in programs" :key="index">
      <div>
        <div>
          <p> <span style="font-weight: bold">{{program.avgScore}}</span> - {{program.sourceCode}} </p>
        </div>
        <div>

        </div>
      </div>
    </div>
  </div>

</template>

<style scoped>

</style>
