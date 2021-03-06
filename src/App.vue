<template>
  <div id="app" :style="theme">
    <GameOver
      :active="gameOver && !seenWinner"
      :close="closeGameOver"
      :winner="winnerText"
    />
    <Settings
      :active="openSettings"
      :close="toggleSettings"
      :select="selectTheme"
      :themes="themes"
      :theme-idx="themeIdx"
    />
    <div class="board">
      <div class="row header">
        <div class="col-flex" :class="{ winner: gameOver && winner !== 'B' }">
          <span>A</span>
          <span>{{ scores.A }}</span>
        </div>
        <div class="col-flex" :class="{ winner: gameOver && winner !== 'A' }">
          <span>{{ scores.B }}</span>
          <span>B</span>
        </div>
      </div>
      <div
        v-for="key in order"
        :key="key"
        class="row"
        :class="{ closed: closed[key] }"
      >
        <div class="col">
          <ScoreButton
            :closed="closed[key]"
            :onClick="() => hit('A', key)"
            :value="playerHits.A[key]"
          />
        </div>
        <div class="col-flex">
          <span class="hit" v-text="key" />
        </div>
        <div class="col">
          <ScoreButton
            :closed="closed[key]"
            :onClick="() => hit('B', key)"
            :value="playerHits.B[key]"
          />
        </div>
      </div>
      <div class="row footer">
        <div class="col-flex act-undo">
          <button @click="undo">UNDO</button>
        </div>
        <div class="col act-setting">
          <button @click="toggleSettings">•••</button>
        </div>
        <div class="col-flex act-new">
          <button @click="clear">NEW</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Vue } from 'vue-property-decorator';
import GameOver from './components/GameOver.vue';
import ScoreButton from './components/ScoreButton.vue';
import Settings from './components/Settings.vue';
import { defaultOrder, defaultPlayerHits } from './defaults';
import {
  Closed,
  ClosedType,
  HitIdType,
  PlayerHits,
  PlayerIdType,
  Scores,
  Timeline,
  TimelineEntry,
  Theme,
  WinningType,
} from './types';
import {
  CLOSED_0,
  CLOSED_1,
  HIT_ID_15,
  HIT_ID_16,
  HIT_ID_17,
  HIT_ID_18,
  HIT_ID_19,
  HIT_ID_20,
  HIT_ID_B,
  PLAYER_ID_A,
  PLAYER_ID_B,
  PLAYER_ID_C,
  PLAYER_ID_D,
  TIE,
  THEMES,
} from './constants';

@Component({
  components: { GameOver, ScoreButton, Settings },
})
export default class App extends Vue {
  version: string = '0.1';
  openSettings: boolean = false;
  order: ReadonlyArray<HitIdType> = defaultOrder;
  playerHits: PlayerHits = defaultPlayerHits;
  seenWinner: boolean = false;
  timeline: Timeline = [];
  themes: Array<Theme> = THEMES;
  themeIdx: number = 0;

  get closed(): Closed {
    const { A, B, C, D } = this.playerHits;
    const check = (k: HitIdType): ClosedType =>
      A[k] >= 3 && B[k] >= 3 ? CLOSED_1 : CLOSED_0;
    return {
      [HIT_ID_20]: check(HIT_ID_20),
      [HIT_ID_19]: check(HIT_ID_19),
      [HIT_ID_18]: check(HIT_ID_18),
      [HIT_ID_17]: check(HIT_ID_17),
      [HIT_ID_16]: check(HIT_ID_16),
      [HIT_ID_15]: check(HIT_ID_15),
      [HIT_ID_B]: check(HIT_ID_B),
    };
  }

  get gameOver(): boolean {
    const closedA = (): boolean => this.closedHits(PLAYER_ID_A).length === 0;
    const closedB = (): boolean => this.closedHits(PLAYER_ID_B).length === 0;
    const closedC = (): boolean => this.closedHits(PLAYER_ID_C).length === 0;
    const closedD = (): boolean => this.closedHits(PLAYER_ID_D).length === 0;
    const allClosed: boolean =
      Object.values(this.closed).filter(a => a === 0).length === 0;
    const winningA = this.winning === PLAYER_ID_A;
    const winningB = this.winning === PLAYER_ID_B;
    const winningC = this.winning === PLAYER_ID_C;
    const winningD = this.winning === PLAYER_ID_D;
    return (
      allClosed ||
      (winningA && closedA()) ||
      (winningB && closedB()) ||
      (winningC && closedC()) ||
      (winningD && closedD())
    );
  }

  get theme(): Theme {
    return this.themes[this.themeIdx || 0];
  }

  get winning(): WinningType {
    const [a, b, c, d] = Object.values(this.scores);
    const max = Math.max(a, b, c, d);
    const winning: Array<PlayerIdType> = [];
    if (this.scores[PLAYER_ID_A] === max) winning.push(PLAYER_ID_A);
    if (this.scores[PLAYER_ID_B] === max) winning.push(PLAYER_ID_B);
    if (this.scores[PLAYER_ID_C] === max) winning.push(PLAYER_ID_C);
    if (this.scores[PLAYER_ID_D] === max) winning.push(PLAYER_ID_D);
    return winning.length > 1 ? TIE : winning[0];
  }

  get winner(): WinningType | null {
    if (!this.gameOver) return null;
    return this.winning;
  }

  get winnerText(): string | null {
    if (!this.winner) return null;
    return this.winner === TIE ? TIE : `Player ${this.winner}`;
  }

  get scores(): Scores {
    let a: number = 0;
    let b: number = 0;
    let c: number = 0;
    let d: number = 0;
    const { order, playerHits } = this;
    order.forEach(key => {
      const val: number = key === HIT_ID_B ? 25 : parseInt(key);
      if (playerHits[PLAYER_ID_A][key] > 3)
        a += (playerHits[PLAYER_ID_A][key] - 3) * val;
      if (playerHits[PLAYER_ID_B][key] > 3)
        b += (playerHits[PLAYER_ID_B][key] - 3) * val;
      if (playerHits[PLAYER_ID_C][key] > 3)
        c += (playerHits[PLAYER_ID_C][key] - 3) * val;
      if (playerHits[PLAYER_ID_D][key] > 3)
        d += (playerHits[PLAYER_ID_D][key] - 3) * val;
    });
    return {
      [PLAYER_ID_A]: a,
      [PLAYER_ID_B]: b,
      [PLAYER_ID_C]: c,
      [PLAYER_ID_D]: d,
    };
  }

  closedHits(playerId: PlayerIdType) {
    return Object.values(this.playerHits[playerId]).filter(a => a < 3);
  }

  @Emit() clear(): void {
    if (window.confirm('Are you sure you want a new game?')) {
      for (let key in this.playerHits) {
        for (let k in this.playerHits[key as PlayerIdType]) {
          this.$set(this.playerHits[key as PlayerIdType], k, 0);
        }
      }
      this.seenWinner = false;
      while (this.timeline.length) this.timeline.pop();
      this.$nextTick(() => this.save());
    }
  }

  @Emit() closeGameOver(): void {
    this.seenWinner = true;
  }

  @Emit() hit(playerId: PlayerIdType, hitId: HitIdType): void {
    const currVal = this.playerHits[playerId][hitId];
    const nextVal = currVal + 1;
    if (this.closed[hitId] || this.gameOver) return;
    const entry: TimelineEntry = { playerId, hitId };
    this.timeline.push(entry);
    this.$set(this.playerHits[playerId], hitId, nextVal);
    this.save();
  }

  @Emit() save(): void {
    const state = {
      playerHits: this.playerHits,
      timeline: this.timeline,
    };
    localStorage.setItem('cricket', JSON.stringify(state));
  }

  @Emit() selectTheme(idx: number): void {
    this.themeIdx = idx;
    localStorage.setItem('theme', idx.toString());
  }

  @Emit() toggleSettings(): void {
    this.openSettings = !this.openSettings;
  }

  @Emit() undo(): void {
    if (!this.timeline.length) return;
    const entry = this.timeline.pop();
    if (!entry) return;
    const hitId: HitIdType = entry.hitId;
    const playerId: PlayerIdType = entry.playerId;
    const nextVal = this.playerHits[playerId][hitId] - 1;
    this.seenWinner = false;
    this.$set(this.playerHits[playerId], hitId, nextVal);
    this.save();
  }

  mounted(): void {
    const version = localStorage.getItem('version');
    if (!version || version !== this.version) {
      localStorage.setItem('version', this.version);
      localStorage.removeItem('cricket');
      localStorage.removeItem('theme');
    }
    const theme = localStorage.getItem('theme');
    if (theme) this.themeIdx = parseInt(theme);

    const saved = localStorage.getItem('cricket');
    if (saved) {
      const { playerHits, timeline } = JSON.parse(saved);
      this.$set(this, 'playerHits', playerHits as PlayerHits);
      this.$set(this, 'timeline', timeline as Timeline);
    }
  }
}
</script>

<style lang="scss">
:root {
  --border: 5px;
  --font-size: 1em;
  --font-size-large: 1.4em;
  --font-size-display-large: 3em;
  --font-size-display-small: 2em;
  --font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
}

html,
body {
  background: black;
  margin: 0;
  height: 100%;
  font-size: var(--font-size);
  touch-action: none;
}
button {
  appearance: none;
  border: none;
  cursor: pointer;
  font-family: var(--font-family);
  font-size: var(--font-size);
  font-weight: bold;
  outline: none;
}
#app {
  -moz-osx-font-smoothing: grayscale;
  -webkit-font-smoothing: antialiased;
  background: var(--color2);
  box-sizing: border-box;
  color: var(--color1);
  font-family: var(--font-family);
  height: 100%;
  @media (min-height: 660px), (min-width: 660px) {
    padding: 1rem;
  }
}
.board {
  display: flex;
  flex-direction: column;
  @media (orientation: landscape) {
    flex-direction: row;
  }
  font-weight: bold;
  height: 100%;
  text-align: center;
  width: 100%;
}
.row {
  background: var(--color2);
  color: var(--color1);
  display: flex;
  @media (orientation: landscape) {
    flex-direction: column;
  }
  justify-content: space-between;
  transition: color 150ms ease-in-out;
  &.closed {
    background: var(--accent);
    color: var(--color2);
    .col-center {
      border-left-color: var(--accent);
      border-right-color: var(--accent);
    }
  }
  &:not(.header):not(.footer) {
    flex: 1;
  }
  @media (orientation: landscape) {
    flex: 1;
  }

  + .row {
    @media (orientation: portrait) {
      border-top: var(--border) solid var(--color3);
    }
    @media (orientation: landscape) {
      border-left: var(--border) solid var(--color3);
    }
  }
  .col,
  .col-flex {
    align-content: center;
    align-items: center;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    justify-content: center;
    @media (orientation: portrait) {
      padding: 0.25rem 0.5rem;
    }
    @media (orientation: landscape) {
      padding: 0.5rem 0.5rem;
    }
    > button,
    > span {
      display: block;
    }
  }
  .col-flex {
    flex: 1;
  }
  span.hit {
    font-size: var(--font-size-large);
    font-weight: 900;
  }
  span.count {
    font-weight: 900;
  }
}

.row.header {
  background: var(--color1);
  color: var(--color2);
  font-weight: 900;
  .col-flex {
    flex-direction: row;
    font-size: var(--font-size-large);
    padding: 1rem;
    &.winner {
      background: var(--accent);
    }
    @media (orientation: landscape) {
      flex-direction: column;
      + .col-flex {
        border-top: var(--border) solid var(--color2);
      }
      span:last-child {
        margin-top: auto;
      }
    }
    @media (orientation: portrait) {
      + .col-flex {
        border-left: var(--border) solid var(--color2);
      }
      span:first-child {
        text-align: left;
      }
      span:last-child {
        margin-left: auto;
        text-align: right;
      }
    }
  }
}

.row.footer {
  background: var(--color1);
  padding: 0;
  > div {
    padding: 0;
  }
  @media (orientation: landscape) {
    > div + div {
      border-top: var(--border) solid var(--color2);
    }
    button {
      height: 100%;
    }
  }
  @media (orientation: portrait) {
    > div + div {
      border-left: var(--border) solid var(--color2);
    }
  }
}

button {
  background: var(--color1);
  color: var(--color2);
  display: block;
  font-size: var(--font-size-large);
  font-weight: 900;
  padding: 1rem 0.5rem;
  width: 100%;
  border: none;
  outline: none;
  margin: 0;
}
</style>
