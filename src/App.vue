<template>
  <main>
    <h1>Mortgage Calculator</h1>
    <div class="io">
      <div class="input">
        <form>
          <div class="row">
            <div class="row-half">
              <label>House Price</label>
              <input type="number" v-model="house_price" @change="calculate" />
            </div>
            <div class="row-half">
              <label>Deposit</label>
              <input type="number" v-model="deposit" @change="calculate" />
            </div>
          </div>
          <div class="row">
            <div class="row-half">
              <label>Repayment Years</label>
              <input type="number" v-model="repayment_years" @change="calculate" />
            </div>
            <div class="row-half">
              <label>Interest Rate</label>
              <input type="number" step="0.1" v-model="interest_rate" @change="calculate" />
            </div>
          </div>
        </form>
      </div>
      <div class="output">
        <table>
          <tr>
            <th>Total To Pay</th>
            <td>£{{ total_to_pay }}</td>
          </tr>
          <tr>
            <th>Total Interest Paid</th>
            <td>£{{ total_interest_to_pay }}</td>
          </tr>
          <tr>
            <th>Monthly Repayments</th>
            <td>£{{ monthly_payment }}</td>
          </tr>
        </table>
      </div>
    </div>
    <div class="io top">
      <div class="chart-holder">
        <Line v-if="chart_setup" id="balance-chart" :options="chart_options" :data="balance_chart" :key="chart_key" />
      </div>
      <div class="save-holder">
        <h2>Saved Calculations <button @click="save">Save Calculation</button></h2>
        <button class="saved-calc" @click="loadSave(save)" v-for="save in saves" :key="save">
          <div><b>House Price: </b>£{{ formatNumber(save.house_price) }}</div>
          <div><b>Deposit:</b> £{{ formatNumber(save.deposit) }}</div>
          <div><b>Repayment Years:</b> {{ save.repayment_years }}</div>
          <div><b>Interest Rate:</b> {{ save.interest_rate }}</div>
        </button>
        <div class="error" v-if="saves.length == 0">No calculations saved</div>
      </div>
    </div>
    <div ref="snackbar" class="snackbar" :class="{ 'active': snackbar.shown }">
      {{ snackbar.message }}
    </div>
  </main>
</template>

<script>
import { Line } from 'vue-chartjs'
import { Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend } from 'chart.js'
ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend)

export default {
  name: 'App',
  components: { Line },
  data() {
    return {
      saves: [],
      deposit: 10000,
      house_price: 200000,
      repayment_years: 20,
      interest_rate: 4,
      total_to_pay: 0,
      total_interest_to_pay: 0,
      monthly_payment: 0,
      snackbar: {
        shown: false,
        message: '',
        timer1: '',
        timer2: '',
        timer3: '',
      },
      chart_setup: false,
      chart_key: 0,
      balance_chart: {
        labels: [],
        datasets: [{
          label: 'Remaining Debt',
          data: [],
          borderColor: '#f87171',
          backgroundColor: 'rgba(248, 113, 113, 0.2)',
          tension: 0.3,
          fill: true,
          pointBackgroundColor: '#f87171',
          pointBorderColor: '#f87171'
        }],
      },
      chart_options: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: { position: 'top' },
          title: {
            display: true,
            text: 'Loan Balance Over Time'
          }
        },
        scales: {
          y: {
            ticks: {
              callback: (value) => '£' + value.toLocaleString()
            }
          }
        }
      }
    }
  },
  mounted() {
    this.calculate();
  },
  methods: {
    showSnackbar(message) {
      clearTimeout(this.snackbar.timer1);
      clearTimeout(this.snackbar.timer2);
      clearTimeout(this.snackbar.timer3);

      this.snackbar.timer1 = setTimeout(() => {
        this.snackbar.shown = false;
        this.snackbar.message = "";
      }, 1);
      this.snackbar.timer2 = setTimeout(() => {
        void this.$refs.snackbar.offsetWidth;
        this.snackbar.shown = true;
        this.snackbar.message = message;
      }, 2);

      this.snackbar.timer3 = setTimeout(() => {
        this.snackbar.shown = false;
        this.snackbar.message = "";
      }, 3000);
    },
    save() {
      this.showSnackbar("Calculation saved!");
      this.saves.push({
        house_price: this.house_price,
        deposit: this.deposit,
        interest_rate: this.interest_rate,
        repayment_years: this.repayment_years
      })
    },
    loadSave(save) {
      this.showSnackbar("Calculation loaded!");
      this.house_price = save.house_price;
      this.deposit = save.deposit;
      this.interest_rate = save.interest_rate;
      this.repayment_years = save.repayment_years;
      this.calculate();
    },
    formatNumber(number) {
      return number.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 });
    },
    calculate() {
      const loan_amount = this.house_price - this.deposit;
      const total_months = this.repayment_years * 12;
      const monthly_rate = this.interest_rate / 100 / 12;
      const monthly_payment = loan_amount * (monthly_rate * Math.pow(1 + monthly_rate, total_months)) /
        (Math.pow(1 + monthly_rate, total_months) - 1);
      var total_interest = 0;
      var balance = loan_amount;

      let balances = [loan_amount];
      let labels = ["Year 0"];

      // Calculate standard mortgage
      for (let month = 1; month <= total_months; month++) {
        const interest = balance * monthly_rate;
        const principal = monthly_payment - interest;
        balance -= principal;
        total_interest += interest;
        if (month % 12 == 0) {
          balances.push(balance.toFixed(2));
          labels.push("Year: " + month / 12);
        }
        if (balance < 0.01) balance = 0;
      }

      this.balance_chart.datasets[0].data = balances.map(Number);
      this.balance_chart.labels = labels;

      this.monthly_payment = this.formatNumber(monthly_payment);
      this.total_interest_to_pay = this.formatNumber(total_interest);
      this.total_to_pay = this.formatNumber(loan_amount + total_interest);
      this.chart_setup = true;
      this.chart_key++;
    }
  }
}
</script>

<style>
#app {
  font-family: 'System UI', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

body {
  margin: 0;
  background: #1a2631;
  color: white;
}

h1,
h2 {
  color: white;
}

h2 {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin: 0px;
  padding: 10px 25px;
  font-size: 18px;
  border-bottom: 3px dotted #576e83;
}

* {
  box-sizing: border-box;
}

main {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.io {
  width: 100%;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  gap: 50px;
  flex-wrap: wrap;
}

.io.top {
  margin-top: 30px;
  align-items: flex-start;
}

.input,
.output {
  width: 50%;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  max-width: 600px;
}

.input {
  justify-content: flex-end;
}

.chart-holder {
  width: 100%;
  max-width: 800px;
  border-radius: 5px;
  background: white;
}

form,
table {
  width: 100%;
}

table {
  background-color: #34495e;
}

table {
  border-collapse: collapse;
  border-radius: 5px;
}

table tr th,
table tr td {
  padding: 11.5px;
  border-bottom: 3px dotted #576e83;
}

table tr:last-child td,
table tr:last-child th {
  border: 0;
}

table tr td {
  color: #ddd;
}

table tr th {
  color: white;
}

.input .row {
  text-align: left;
  width: 100%;
  margin-top: 10px;
  display: flex;
  gap: 20px;
}

.input .row .row-half {
  width: 50%;
}

.input .row input {
  width: 100%;
  padding: 15px 20px;
  border-radius: 5px;
  background: #2c3e50;
  border: 0px;
  outline: 0;
  color: #ddd;
}

.input .row label {
  display: block;
  font-weight: 600;
  margin-bottom: 5px;
  color: white;
}

.save-holder {
  width: 100%;
  max-width: 400px;
  max-height: 400px;
  overflow-y: auto;
  background-color: #34495e;
  border-radius: 5px;
}

.save-holder .error {
  color: #ddd;
  padding: 20px 25px;
  text-align: left;
}

.saved-calc {
  width: 100%;
  border: 0;
  background: transparent;
  cursor: pointer;
  color: #ddd;
  padding: 20px 25px;
  border-bottom: 3px dotted #576e83;
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.saved-calc:last-of-type {
  border-bottom: 0;
}

button {
  background-color: transparent;
  text-transform: uppercase;
  font-size: 10px;
  border: 2px solid #ddd;
  padding: 5px 10px;
  border-radius: 5px;
  color: #ddd;
  cursor: pointer;
}

button:hover {
  opacity: 0.8;
}

.snackbar {
  position: fixed;
  bottom: -100px;
  opacity: 0;
  background-color: #27ae60;
  color: white;
  border-radius: 5px;
  width: 400px;
  padding: 15px;
}

.snackbar.active {
  animation: snackbar 3000ms ease forwards;
}

@keyframes snackbar {
  0% {
    bottom: -100px;
    opacity: 0;
  }

  5% {
    bottom: 10px;
    opacity: 1;
  }

  95% {
    bottom: 10px;
    opacity: 1;
  }

  100% {
    bottom: -100px;
    opacity: 0;
  }
}

#balance-chart {
  min-height: 400px;
}

@media screen and (max-width: 1000px) {
  main {
    width: calc(100% - 20px);
    margin: 0 auto;
    height: auto;
  }

  .input,
  .output {
    width: 100%;
  }

  .io {
    gap: 20px;
  }
}
</style>
