<template>
  <div class="rating container-fluid">
    <nav class="navbar navbar-expand-sm navbar-expand-lg bg-dark navbar-dark">
        <ul class="navbar-nav">
            <li class="nav-item active">
            <a class="nav-link" href="#">Active</a>
            </li>
            <li class="nav-item">
            <a class="nav-link" href="#">Link</a>
            </li>
            <li class="nav-item">
            <a class="nav-link" href="#">Link</a>
            </li>
            <li class="nav-item">
            <a class="nav-link disabled" href="#">Disabled</a>
            </li>
        </ul>
    </nav>
    <div class="jumbotron bg-dark text-white">
        <h1 v-once>{{content.pageTitle}}</h1> 
        <p v-once>{{ content.pageDescription}}</p> 
    </div>
      <form action="#">
        <div class="form-group">
            <label for="mortgageAmount" v-once>{{ content.principalLabel}}</label>
            <input type="text" v-model.lazy="form.outstandingMortgageDebt" class="form-control" id="mortgageAmount">
        </div>
        <div class="form-group">
            <label for="interestRate" v-once>{{ content.interestLabel }}</label>
            <input type="text" v-model.lazy="form.mortgageRate" class="form-control" id="interestRate">
        </div>
        <div class="form-group">
            <label for="term" v-once>{{content.termLabel}}</label>
            <input type="text" v-model.lazy="form.mortgageTerm" class="form-control" id="term">
        </div>
      </form>

      <button v-on:click="calculate" class="btn btn-dark" v-once>{{ content.calculateBtnText }}</button>

      <div  v-if="hasCalculationBeenPerformed" class="jumbotron summary" >
        <div class="monthly" >
            <h2>Summary</h2>
            <p>Monthly Payment - £{{ this.monthlyPayment }}</p>
            <p>Total interest over term - £{{ totalInterest }}</p>
        </div>

        <div class="graphArea">
            <h2 v-once>{{content.graphTitle}}</h2>
            <line-graph :width="300" :height="300" :chart-data="chartData" ></line-graph>
        </div>
        <div>
            <table class="table table-dark table-hover">
                <thead>
                    <tr>
                        <th>Year</th>
                        <th>Mortgage Debt</th>
                        <th>Capital Repayment</th>
                        <th>Interest Paid</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="i in items" v-bind:key="i.year">
                        <td>{{i.year}}</td>
                        <td>{{i.amount}}</td>
                        <td v-if="i.capitalRepayment > 0">{{i.capitalRepayment}}</td>
                        <td v-if="i.capitalRepayment == 0">-</td>
                        <td>{{i.interestPaid}}</td>
                    </tr>
                </tbody>
            </table>
        </div>
      </div>

  </div>
</template>

<script>
import LineGraph from '../components/Linegraph'

export default {
  components: { LineGraph },
  data () {
    return {
      monthlyPayment: 0,
      content: {
        principalLabel: 'Principal:',
        pageTitle: 'Mortgage Calculator',
        pageDescription: 'Model your mortgage and find the right term for you...',
        interestLabel: 'Annual interest rate:',
        termLabel: 'Term:',
        calculateBtnText: 'Calculate',
        graphTitle: 'Mortgage principal over time'
      },
      form: {
        outstandingMortgageDebt: 100000,
        mortgageRate: 3,
        mortgageTerm: 35
      },
      totalInterest: 0,
      paymentSchedule: 'Yearly',
      hasCalculationBeenPerformed: false,
      selected: 'Y',
      options: [
        { text: 'Yearly', value: 'Y' },
        { text: 'Monthly', value: 'M' },
        { text: 'Fortnightly', value: 'F' }
      ],
      chartData: null,
      items: [
        {
          year: 1,
          amount: 100000,
          capitalRepayment: 0,
          interestPaid: 0
        }
      ]
    }
  },
  methods: {
    calculate: function (event) {
      var term = this.form.mortgageTerm
      var debt = this.form.outstandingMortgageDebt
      var rate = this.form.mortgageRate

      var monthlyInterestRate = (rate / 100) / 12
      var numberOfMonthlyPayments = term * 12

      // see method 2 https://www.wikihow.com/Calculate-Mortgage-Payments
      var topOfMortgageEquation = monthlyInterestRate * Math.pow(1 + monthlyInterestRate, numberOfMonthlyPayments)
      var bottomOfMortgageEquation = Math.pow(1 + monthlyInterestRate, numberOfMonthlyPayments) - 1
      var equationAnswer = topOfMortgageEquation / bottomOfMortgageEquation

      this.monthlyPayment = Math.ceil(debt * equationAnswer)

      this.totalInterest = (this.monthlyPayment * numberOfMonthlyPayments) - debt
      
      this.getYearlyDebt()

    },
    getYearlyDebt: function () {
      var term = this.form.mortgageTerm
      var debt = this.form.outstandingMortgageDebt
      var rate = this.form.mortgageRate

      // see http://www.mathshelper.co.uk/Mortgage%20Repayment%20Formula%20Derivation.pdf
      var topOfMortgageEquation = debt * rate
      var bottomOfMortgageEquation = 100 * (Math.pow(1 + (rate / 100), term) - 1)
      var equationAnswer = topOfMortgageEquation / bottomOfMortgageEquation

      this.items.length = 0

      this.items.push({year: 0, amount: debt, capitalRepayment: 0})
      debt = Math.ceil(debt - equationAnswer)
      this.items.push({year: 1, amount: Math.ceil(debt), capitalRepayment: Math.round(equationAnswer)})

      var adjustedRate = 1 + (rate / 100)
      var result = 0

      for (var i = 2; i < term; i++) {
        result = Math.pow(adjustedRate, i - 1) * equationAnswer
        debt = debt - result
        this.items.push({year: i, amount: Math.ceil(debt), capitalRepayment: Math.round(result)})
      }

      this.items.push({year: term, amount: 0, capitalRepayment: 0})
      this.populateChartData()
    },

    populateChartData: function () {
      var labelArr = []
      var dataArr = []

      for (var i = 0; i < this.items.length; i++) {
        labelArr.push(this.items[i].year)
        dataArr.push(this.items[i].amount)
      }

      this.hasCalculationBeenPerformed = true

      this.chartData = {
        labels: labelArr,
        datasets: [{
          label: 'Principal',
          backgroundColor: '#f87979',
          data: dataArr
        }
        ]
      }
    }
  }
}

</script>

<style scoped>
  @import '../assets/bootstrap.css';
  @import '../assets/calculator.css';
</style>
