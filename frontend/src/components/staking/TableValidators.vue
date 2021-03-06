<template>
  <div>
    <table class="data-table card-white">
      <thead>
        <PanelSort
          :sort="sort"
          :properties="properties"
          :show-on-mobile="showOnMobile"
        />
      </thead>
      <tbody
        is="transition-group"
        v-infinite-scroll="loadMore"
        infinite-scroll-distance="400"
        name="flip-list"
      >
        <LiValidator
          v-for="(validator, index) in showingValidators"
          :key="validator.operator_address"
          :index="index"
          :validator="validator"
          :show-on-mobile="showOnMobile"
        />
      </tbody>
    </table>
  </div>
</template>

<script>
import { mapGetters, mapState } from "vuex"
import orderBy from "lodash.orderby"
import LiValidator from "staking/LiValidator"
import PanelSort from "staking/PanelSort"
import { expectedReturns } from "scripts/returns"
export default {
  name: `table-validators`,
  components: {
    LiValidator,
    PanelSort
  },
  props: {
    validators: {
      type: Array,
      required: true
    },
    showOnMobile: {
      type: String,
      default: () => "returns"
    }
  },
  data: () => ({
    query: ``,
    sort: {
      property: `expectedReturns`,
      order: `desc`
    },
    showing: 15,
    rollingWindow: 10000 // param of slashing period
  }),
  computed: {
    ...mapState([`distribution`, `pool`, `session`, "delegates"]),
    ...mapState({
      annualProvision: state => state.minting.annualProvision
    }),
    ...mapGetters([`committedDelegations`, `bondDenom`, `lastHeader`]),
    enrichedValidators(
      {
        validators,
        pool,
        annualProvision,
        committedDelegations,
        session,
        distribution
      } = this
    ) {
      return validators.map(v => {
        const delegation = this.delegates.delegates.find(
          d => d.validator_address === v.operator_address
        )

        return Object.assign({}, v, {
          small_moniker: v.moniker.toLowerCase(),
          my_delegations: delegation ? delegation.amount : 0,
          // my_delegations:
          //   session.signedIn && committedDelegations[v.operator_address] > 0
          //     ? committedDelegations[v.operator_address]
          //     : 0,
          rewards:
            session.signedIn && distribution.rewards[v.operator_address]
              ? distribution.rewards[v.operator_address][this.bondDenom]
              : 0,
          expectedReturns: annualProvision
            ? expectedReturns(
                v,
                parseInt(pool.pool.bonded_tokens),
                parseFloat(annualProvision)
              )
            : undefined
        })
      })
    },
    sortedEnrichedValidators() {
      return orderBy(
        this.enrichedValidators.slice(0),
        [this.sort.property],
        [this.sort.order]
      )
    },
    showingValidators() {
      return this.sortedEnrichedValidators.slice(0, this.showing)
    },
    properties() {
      return [
        {
          title: `Name`,
          value: `small_moniker`,
          tooltip: `The validator's moniker`
        },
        {
          title: `Total ONE staked`,
          value: `total`,
          tooltip: `Total ONE staked`
        },
        {
          title: `Voting Power`,
          value: `avg_voting_power`,
          tooltip: `Percentage of voting shares`
        }
      ]
    }
  },
  watch: {
    "sort.property": function() {
      this.showing = 15
    },
    "sort.order": function() {
      this.showing = 15
    }
  },
  mounted() {
    this.$store.dispatch(`getPool`)
    this.$store.dispatch(`getRewardsFromMyValidators`)
    this.$store.dispatch(`getMintingParameters`)
  },
  methods: {
    loadMore() {
      this.showing += 10
    }
  }
}
</script>
<style scoped>
@media screen and (max-width: 550px) {
  .data-table td {
    overflow: hidden;
  }

  .data-table__row__info {
    max-width: 22rem;
  }
}

.flip-list-move {
  transition: transform 0.3s;
}
</style>