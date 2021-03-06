<template>
  <TmPage
    :managed="true"
    :loading="loading"
    :loaded="!loading"
    :error="validator.error"
    :data-empty="!validator.operator_address"
    :hide-header="true"
    data-title="Validator"
    class="small"
  >
    <template v-if="validator.operator_address" slot="managed-body">
      <div class="status-container">
        <span :class="status | toLower" class="validator-status">
          {{
          status
          }}
        </span>
        <span v-if="status_detailed" class="validator-status-detailed">
          {{
          status_detailed
          }}
        </span>
      </div>
      <tr class="li-validator">
        <td class="data-table__row__info">
          <Avatar
            v-if="!validator.avatarUrl"
            class="li-validator-image"
            alt="generic geometric symbol - generated avatar from address"
            :address="validator.operator_address"
          />
          <img
            v-else-if="validator.avatarUrl"
            :src="validator.avatarUrl"
            :alt="`validator logo for ` + validator.moniker"
            class="li-validator-image"
          />
          <div class="validator-info">
            <h3 class="li-validator-name">{{ validator.moniker }}</h3>
            <div v-if="myDelegation">
              <h4>{{ myDelegation }}</h4>
              <h5 v-if="rewards">{{ 0 | atoms | shortDecimals | noBlanks }}</h5>
            </div>
          </div>
        </td>
      </tr>

      <div class="button-container">
        <TmBtn id="delegation-btn" value="Stake" @click.native="onDelegation" />
        <TmBtn
          id="undelegation-btn"
          :disabled="!myDelegation"
          value="Unstake"
          type="secondary"
          @click.native="onUndelegation"
        />
      </div>

      <div class="card-white">
        <ul class="row">
          <li class="column">
            <h4>Description</h4>
            <span>{{ validator.details | noBlanks }}</span>
          </li>
          <li class="column">
            <h4>Website</h4>
            <span v-if="website !== `--`">
              <a
                id="validator-website"
                :href="website"
                target="_blank"
                rel="nofollow noreferrer noopener"
              >{{ website }}</a>
            </span>
            <span v-else id="validator-website">{{ website | noBlanks }}</span>
          </li>
          <li class="column">
            <h4>Validator Address</h4>
            <span>
              <Bech32 :address="validator.operator_address" />
            </span>
          </li>
        </ul>

        <ul class="row">
          <li>
            <h4>Voting Power / Total Stake</h4>
            <span id="page-profile__power">
              {{ validator.avg_voting_power | percent }} /
              {{ (validator.total_effective_stake / 1e18) | shortDecimals }}
            </span>
          </li>
          <li>
            <h4>Self Stake</h4>
            <span id="page-profile__self-bond">{{ selfBondPercent }} / {{ selfBondAmount }}</span>
          </li>
          <li>
            <h4>Min Self Delegation</h4>
            <span id="page-profile__min_self_delegation">
              {{
              (validator.min_self_delegation / 1e18) | shortDecimals
              }}
            </span>
          </li>
          <li>
            <h4>Max Total Delegation</h4>
            <span id="page-profile__max_total_delegation">
              {{
              (validator.max_total_delegation / 1e18) | shortDecimals
              }}
            </span>
          </li>
          <li>
            <h4>Validator Since</h4>
            <span>Block #{{ validator.creation_height }}</span>
          </li>
          <li>
            <h4>Uptime</h4>
            <span id="page-profile__uptime">
              {{
              validator.uptime_percentage | percent
              }}
            </span>
          </li>
          <li>
            <h4>Current Commission Rate</h4>
            <span>{{ validator.rate | percent }}</span>
          </li>
          <li>
            <h4>Max Commission Rate</h4>
            <span>{{ validator.max_rate | percent }}</span>
          </li>
          <li>
            <h4>Max Daily Commission Change</h4>
            <span>{{ validator.max_change_rate | percent }}</span>
          </li>
          <li>
            <h4>Last Commission Change</h4>
            <span>Block #{{ validator.update_height }}</span>
          </li>
        </ul>
      </div>

      <DelegationModal
        ref="delegationModal"
        :from-options="delegationTargetOptions()"
        :to="validator.operator_address"
        :validator="validator"
        :denom="bondDenom"
      />
      <UndelegationModal
        ref="undelegationModal"
        :maximum="Number(myBond)"
        :to="session.signedIn ? session.address : ``"
        :validator="validator"
        :denom="bondDenom"
        @switchToRedelegation="onDelegation({ redelegation: true })"
      />
    </template>
    <template v-else>
      <div slot="title">Validator Not Found</div>
      <div slot="subtitle">
        Please visit the
        <router-link to="/validators/">Validators</router-link>page to view all
        validators
      </div>
    </template>
  </TmPage>
</template>

<script>
import moment from "moment"
import { calculateTokens } from "scripts/common"
import { mapState, mapGetters } from "vuex"
import { atoms, viewDenom, shortDecimals, percent, uatoms } from "scripts/num"
import { formatBech32 } from "src/filters"
import { expectedReturns } from "scripts/returns"
import TmBtn from "common/TmBtn"
import DelegationModal from "src/ActionModal/components/DelegationModal"
import UndelegationModal from "src/ActionModal/components/UndelegationModal"
import Avatar from "common/Avatar"
import Bech32 from "common/Bech32"
import TmPage from "common/TmPage"
import isEmpty from "lodash.isempty"
import { fetchValidatorByAddress } from "../../mock-service"

export default {
  name: `page-validator`,
  components: {
    Bech32,
    DelegationModal,
    UndelegationModal,
    Avatar,
    TmBtn,
    TmPage
  },
  filters: {
    atoms,
    shortDecimals,
    percent,
    toLower: text => text.toLowerCase(),
    // empty descriptions have a strange '[do-not-modify]' value which we don't want to show
    noBlanks: function(value) {
      if (!value || value === `[do-not-modify]`) return `--`
      return value
    }
  },
  props: {
    showOnMobile: {
      type: String,
      default: () => "returns"
    }
  },
  data: () => ({
    loading: true,
    validator: {}
  }),
  computed: {
    ...mapState([
      `delegates`,
      `delegation`,
      `distribution`,
      `pool`,
      `session`,
      `connection`
    ]),
    ...mapState({
      annualProvision: state => state.minting.annualProvision
    }),
    ...mapGetters([
      `lastHeader`,
      `bondDenom`,
      `committedDelegations`,
      `liquidAtoms`,
      `connected`
    ]),
    networkId() {
      return this.connection.networkConfig.id
    },
    selfBondPercent() {
      return percent(this.validator.self_stake / this.validator.total_stake)
    },
    selfBondAmount() {
      return shortDecimals(this.validator.self_stake / 1e18)
    },
    myBond() {
      if (!this.validator.operator_address) return 0

      const delegator = this.delegates.delegates.find(
        d => d.validator_address === this.validator.operator_address
      )

      return atoms(
        calculateTokens(this.validator, delegator ? delegator.amount : 0)
      )
    },
    myDelegation() {
      const { bondDenom, myBond } = this
      const myDelegation = shortDecimals(myBond)
      const myDelegationString = `${myDelegation} ${viewDenom(bondDenom)}`
      return Number(myBond) === 0 ? null : myDelegationString
    },
    returns() {
      return expectedReturns(
        this.validator,
        parseInt(this.pool.pool.bonded_tokens),
        parseFloat(this.annualProvision)
      )
    },
    status() {
      if (
        this.validator.jailed ||
        this.validator.tombstoned ||
        this.validator.status === 0 ||
        this.validator.active === false
      )
        return `Inactive`
      return `Active`
    },
    status_detailed() {
      if (this.validator.jailed) return `Temporally banned from the network`
      if (this.validator.tombstoned) return `Banned from the network`
      if (this.validator.status === 0) return `Banned from the network`
      return false
    },
    website() {
      let url = this.validator.website

      if (!url || url === "[do-not-modify]") {
        return ""
      } else if (!url.match(/http[s]?/)) {
        url = `https://` + url
      }
      return url
    },
    rewards() {
      const { session, bondDenom, distribution, validator } = this
      if (!session.signedIn) {
        return null
      }

      const validatorRewards = distribution.rewards[validator.operator_address]
      return validatorRewards ? validatorRewards[bondDenom] : 0
    }
  },
  watch: {
    networkId: async function() {
      this.loading = true

      if (this.connection.networkConfig.id) {
        this.validator = await fetchValidatorByAddress(
          this.connection.networkConfig.id,
          this.$route.params.validator
        )
      }

      this.loading = false
    },
    myBond: {
      handler(myBond) {
        if (myBond > 0) {
          // this.$store.dispatch(
          //   `getRewardsFromValidator`,
          //   this.$route.params.validator
          // )
        }
      }
    },
    "validator.operator_address": {
      immediate: true,
      handler(operator_address) {
        if (!operator_address) return
        this.$store.dispatch(`getSelfBond`, this.validator)
      }
    },
    lastHeader: {
      immediate: true,
      handler(newHeader) {
        const waitTwentyBlocks = Number(newHeader.height) % 20 === 0
        if (
          this.session.signedIn &&
          waitTwentyBlocks &&
          this.$route.name === `validator` &&
          this.delegation.loaded &&
          Number(this.myBond) > 0
        ) {
          // this.$store.dispatch(
          //   `getRewardsFromValidator`,
          //   this.$route.params.validator
          // )
        }
      }
    }
  },

  async mounted() {
    if (this.connection.networkConfig.id) {
      this.validator = await fetchValidatorByAddress(
        this.connection.networkConfig.id,
        this.$route.params.validator
      )

      this.loading = false
    }
  },
  methods: {
    shortDecimals,
    atoms,
    uatoms,
    percent,
    moment,
    onDelegation(options) {
      this.$refs.delegationModal.open(options)
    },
    onUndelegation() {
      this.$refs.undelegationModal.open()
    },
    delegationTargetOptions(
      { session, liquidAtoms, committedDelegations, $route, delegates } = this
    ) {
      if (!session.signedIn) return []

      //- First option should always be your wallet (i.e normal delegation)
      const myWallet = [
        {
          address: session.address,
          maximum: Math.floor(liquidAtoms),
          key: `My Wallet - ${formatBech32(session.address, false, 20)}`,
          value: 0
        }
      ]
      const bondedValidators = Object.keys(committedDelegations)
      if (isEmpty(bondedValidators)) {
        return myWallet
      }
      //- The rest of the options are from your other bonded validators
      //- We skip the option of redelegating to the same address
      const redelegationOptions = bondedValidators
        .filter(address => address != $route.params.validator)
        .reduce((validators, address) => {
          const delegate = delegates.delegates.find(function(validator) {
            return validator.operator_address === address
          })
          return validators.concat({
            address: address,
            maximum: Math.floor(committedDelegations[address]),
            key: `${delegate.description.moniker} - ${formatBech32(
              delegate.operator_address,
              false,
              20
            )}`,
            value: validators.length + 1
          })
        }, [])
      return myWallet.concat(redelegationOptions)
    }
  }
  // apollo: {
  //   validator: {
  //     query() {
  //       /* istanbul ignore next */
  //       return ValidatorProfile(this.connection.network)
  //     },
  //     variables() {
  //       /* istanbul ignore next */
  //       return {
  //         address: this.$route.params.validator
  //       }
  //     },
  //     update(data) {
  //       /* istanbul ignore next */
  //       return ValidatorResult(this.connection.network)(data)
  //     }
  //   }
  // }
}
</script>
<style scoped>
.li-validator {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border-bottom: 1px solid var(--bc-dim);
  color: var(--txt-black);
}

.li-validator-image {
  border-radius: 0.25rem;
  height: 4rem;
  width: 4rem;
  border: 1px solid var(--bc-dim);
}

.li-validator-name {
  color: var(--txt);
  font-size: var(--h1);
  line-height: 2rem;
  font-weight: 500;
}

.validator-info {
  display: flex;
  flex-direction: column;
  padding-left: 1rem;
  text-overflow: ellipsis;
}

.li-validator h4,
.li-validator h5 {
  font-size: var(--sm);
  display: inline-block;
}

.li-validator h5 {
  padding-left: 0.5rem;
  color: var(--success);
}

span {
  font-size: 12px;
  line-height: normal;
}

.bech32-address {
  font-size: 12px;
  line-height: normal;
}

.button-container {
  display: flex;
  align-items: flex-end;
  padding: 0.5rem 1rem;
  border-bottom: 1px solid var(--bc-dim);
}

.button-container button:first-child {
  margin-right: 0.5rem;
}

.status-container {
  padding: 1rem 1rem 0;
}

.validator-status {
  text-transform: uppercase;
  font-size: 10px;
  font-weight: 600;
  border: 2px solid;
  padding: 2px 4px;
  border-radius: 0.25rem;
}

.validator-status.inactive {
  color: red;
  border-color: red;
}

.validator-status.active {
  color: var(--success);
  border-color: var(--success);
}

.validator-status-detailed {
  display: block;
  margin-top: 0.4rem;
  color: var(--warning);
  font-size: 0.8rem;
}
</style>
<style>
@media screen and (max-width: 667px) {
  .button-container {
    width: 100%;
    padding: 1rem;
  }

  .button-container button {
    width: 50%;
  }
}
</style>
