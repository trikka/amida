<template>
  <v-template>
    <v-row>
      <v-col>
        <v-text-field
          label="参加者は？"
          suffix="人"
          type="number"
          :rules="validater"
          @input="value => this.person = Number(value)">
        </v-text-field>
      </v-col>
      <v-col cols="2">
        <v-btn
          block
          height="100%"
          :disabled="isError() || !completeInput()"
          @click="drawAmidaBridge">あみだ生成</v-btn>
      </v-col>
    </v-row>
    <v-row v-if="!isError()">
      <v-col v-for="person of persons">
        <v-text-field
          :color="person.color"
          :background-color="person.bgColor"
          :label="person.label"
          suffix="さん"
          v-model="person.value"
          :readonly="noEdit">
        </v-text-field>
      </v-col>
    </v-row>
    <v-row v-if="amidaArray && amidaArray.length > 0">
      <v-col>
        <table class="amida">
          <tr v-for="row of amidaArray">
            <td v-for="col of row" :class="{ bridge: col.isBridge }"></td>
          </tr>
        </table>
      </v-col>
    </v-row>
    <v-row v-if="!isError()">
      <v-col v-for="choice of choices">
        <v-text-field
          :background-color="choice.bgColor"
          :label="choice.label"
          v-model="choice.value"
          :readonly="noEdit">
        </v-text-field>
      </v-col>
    </v-row>
  </v-template>
</template>

<style scoped>
table.amida {
  width: 100%;
  height: 600px;
  display: flex;
  flex-flow: column;
}
table.amida tr {
  flex-grow: 1;
  display: flex;
}
table.amida tr td {
  flex-grow: 1;
}
table.amida tr td:nth-child(odd) {
  border-right: 2px solid black;
}
table.amida tr td.bridge,
table.amida tr td.bridge + td {
  border-bottom: 1px solid black;
}
</style>

<script>
export default {
  name: 'IndexPage',
  data () {
    return {
      person: 0,
      validater: [
        this.isInputNumber,
        this.isInteger,
        this.isInRange
      ],
      amidaArray: [],
      persons: [],
      choices: [],
      noEdit: false,
      createdColors: []
    }
  },
  watch: {
    person: function (person) {
      if (!this.isError()) {
        this.noEdit = false;
        this.amidaArray = this.createAmidaArray(person);
        this.persons = [...Array(person)].map((_, i) => ({ id: i, label: '参加者' + (i + 1), color: this.getColor() }));
        this.choices = [...Array(person)].map((_, i) => ({ label: '選択肢' + (i + 1) }));
      }
    }
  },
  methods: {
    inputPerson (event) {
      this.person = Number(event.target.value);
    },
    isInputNumber (value) {
      if (isNaN(value)) {
        return '数字を入力してください'
      }
    },
    isInteger (value) {
      if (!Number.isInteger(Number(value))) {
        return '整数を入力してください'
      }
    },
    isInRange (value) {
      if (value < 2 || value > 10) {
        return '2以上、10以下で入力してください'
      }
    },
    isError () {
      return Boolean(this.validater.find(v => v(this.person)));
    },
    completeInput () {
      return this.persons.every(({value}) => value) && this.choices.every(({value}) => value);
    },
    createAmidaArray (person = this.person) {
      // あみだ用配列の生成
      const amidaArray = [];
      // 参加人数に応じたあみだ配列を生成
      for (let row = 0; row < person * 3; row++) {
        const rowArray = [];
        for (let col = 0; col < person * 2; col++) {
          rowArray.push({});
        }
        amidaArray.push(rowArray);
      }
      return amidaArray;
    },
    drawAmidaBridge () {
      // あみだ参加者・選択肢のロック
      this.noEdit = true;
      const personIds = this.persons.map(({id}) => id);
      // 横線情報を含むあみだ配列を生成して上書きする
      this.amidaArray = this.createAmidaArray().map((row, rowIdx) => {
        return row.map((col, colIdx) => {
          // 横線を引ける範囲か判定
          if (0 < colIdx && colIdx < row.length - 1 && rowIdx < this.amidaArray.length - 1) {
            // 横線が連続しないように、一つ左の範囲に横線を引いていないか判定
            if (colIdx == 1 || colIdx % 2 == 1 && !row[colIdx - 2].isBridge) {
              // 1/2で横線を引く
              col.isBridge = Math.random() > 0.5;
              if (col.isBridge) {
                // 横線を引く場合、参加者の情報を入れ替える
                const personIdx = (colIdx - 1) / 2,
                  leftColumnPerson = personIds[personIdx],
                  rightColumnPerson = personIds[personIdx + 1];
                personIds[personIdx] = rightColumnPerson;
                personIds[personIdx + 1] = leftColumnPerson;
              }
            }
          }
          return col;
        });
      });
      this.persons = this.persons.map(({id, label, value, color}) => ({
        id,
        label,
        value,
        bgColor: color
      }));
      this.choices = this.choices.map((choice, idx) => {
        console.log(personIds, personIds[idx])
        const targetPerson = this.persons.find(({id}) => id === personIds[idx]);
        console.log(targetPerson, this.persons);
        choice.label = targetPerson.value + ' さん';
        choice.bgColor = targetPerson.bgColor;
        return choice;
      });
    },
    getColor () {
      const color = [
        Math.trunc(Math.random() * 255),
        Math.trunc(Math.random() * 255),
        Math.trunc(Math.random() * 255),
      ]
      if (isMonotone(color) || isSimilar(color, this.createdColors)) {
        return this.getColor();
      }
      this.createdColors.push(color);
      return `rgb(${color.join()})`;

      function isMonotone ([r, g, b]) {
        const colorIndex = r * g * b; // color.reduce((p, c) => p * c, 1);
        const [min, , max] = [r, g, b].sort((a, b) => a - b);
        return colorIndex < 100000 // 黒に近い
          || colorIndex > 10000000 // 白に近い
          || max - min < 30; // 彩度が低い
      }
      function isSimilar ([red, green, blue], createdColors) {
        return createdColors.some(([r, g, b]) => diff(red, r) * diff(green, g) * diff(blue, b) < 27000);
        function diff(a, b) {
          return a === b ? 1 : Math.abs(a - b)
        }
      }
    },
  }
}
</script>
