<template lang='pug'>
  li(data-test='proposalItem')
    .c-item
      i(:class='iconClass')
      .c-main
        p.has-text-bold(data-test='typeDescription') {{typeDescription}}
        p.has-text-1(
          :class='{ "has-text-danger": proposal.status === statuses.STATUS_FAILED, "has-text-success": proposal.status === statuses.STATUS_PASSED }'
          data-test='statusDescription'
          ) {{statusDescription}}
      .ctas
        proposal-vote-options(
          v-if='proposal.status === statuses.STATUS_OPEN'
          :proposalHash='proposalHash'
        )
    p.c-sendLink(v-if='invitationLink' data-test='sendLink')
      i18n(
        :args='{ user: proposal.data.proposalData.member}'
      ) Please send the following link to {user} so they can join the group:
      | &nbsp;
      a.link(:href='invitationLink') {{invitationLink}}
</template>

<script>
import { mapGetters, mapState } from 'vuex'
import L from '@view-utils/translations.js'
import {
  PROPOSAL_INVITE_MEMBER,
  PROPOSAL_REMOVE_MEMBER,
  PROPOSAL_GROUP_SETTING_CHANGE,
  PROPOSAL_PROPOSAL_SETTING_CHANGE,
  PROPOSAL_GENERIC,
  STATUS_OPEN,
  STATUS_PASSED,
  STATUS_FAILED,
  STATUS_EXPIRED,
  STATUS_CANCELLED,
  buildInvitationUrl
} from '@model/contracts/voting/proposals.js'
import ProposalVoteOptions from '@containers/proposals/ProposalVoteOptions.vue'

export default {
  name: 'ProposalItem',
  props: {
    proposalHash: String
  },
  components: {
    ProposalVoteOptions
  },
  computed: {
    ...mapGetters([
      'currentGroupState',
      'groupMembersCount'
    ]),
    ...mapState(['currentGroupId']),
    statuses () {
      return { STATUS_OPEN, STATUS_PASSED, STATUS_FAILED, STATUS_EXPIRED, STATUS_CANCELLED }
    },
    proposal () {
      return this.currentGroupState.proposals[this.proposalHash]
    },
    proposalType () {
      return this.proposal.data.proposalType
    },
    typeDescription () {
      return {
        [PROPOSAL_INVITE_MEMBER]: () => L('Add {user} to group.', {
          user: this.proposal.data.proposalData.member
        }),
        [PROPOSAL_REMOVE_MEMBER]: () => L('Remove {user} from group.', {
          user: this.proposal.data.proposalData.member
        }),
        [PROPOSAL_GROUP_SETTING_CHANGE]: () => L('TODO: Change [setting] from [current] to [new-value]', {}),
        [PROPOSAL_PROPOSAL_SETTING_CHANGE]: () => L('TODO: Change [rule setting] from [current] to [new-value]', {}),
        [PROPOSAL_GENERIC]: () => L('TODO: Change [generic] from [current] to [new-value]', {})
      }[this.proposalType]()
    },
    statusDescription () {
      switch (this.proposal.status) {
        case STATUS_OPEN: {
          const votesCount = Object.keys(this.proposal.votes).length

          return L('{count} out of {total} members voted.', {
            count: votesCount,
            total: this.groupMembersCount
          })
        }
        case STATUS_FAILED: {
          return L('Proposal refused.')
        }
        case STATUS_CANCELLED: {
          return L('Proposal cancelled.')
        }
        case STATUS_PASSED: {
          return L('Proposal accepted!')
        }
        default:
          return `TODO status: ${this.proposal.status}`
      }
    },
    iconClass () {
      const type = {
        [PROPOSAL_INVITE_MEMBER]: 'icon-user-plus',
        [PROPOSAL_REMOVE_MEMBER]: 'icon-user-times',
        [PROPOSAL_GROUP_SETTING_CHANGE]: 'icon-users-cog',
        [PROPOSAL_PROPOSAL_SETTING_CHANGE]: 'icon-chart-pie',
        [PROPOSAL_GENERIC]: 'icon-poll'
      }

      const status = {
        [STATUS_OPEN]: 'has-background-primary has-text-primary',
        [STATUS_PASSED]: 'icon-check has-background-success has-text-success',
        [STATUS_FAILED]: 'icon-times has-background-danger has-text-danger',
        [STATUS_CANCELLED]: 'has-background-general has-text-1'
      }

      if ([STATUS_PASSED, STATUS_FAILED].includes(this.proposal.status)) {
        // Show the status icon, no matter the proposal type
        return `${status[this.proposal.status]} c-icon`
      }

      return `${type[this.proposalType]} ${status[this.proposal.status]} c-icon`
    },
    invitationLink () {
      if (this.proposalType === PROPOSAL_INVITE_MEMBER && this.proposal.status === STATUS_PASSED) {
        const secret = this.proposal.payload.inviteSecret
        if (this.currentGroupState.invites[secret].status === 'valid') {
          return buildInvitationUrl(this.currentGroupId, this.proposal.payload.inviteSecret)
        }
      }
      return false
    }
  }
}
</script>

<style lang="scss" scoped>
@import "../../../assets/style/_variables.scss";

$spaceVertical: $spacer-sm*3;

.c-item {
  display: flex;
  align-items: flex-start;
  margin-top: $spaceVertical;
}

.c-main {
  flex-grow: 1;
}

.c-icon {
  width: 2rem;
  height: 2rem;
  margin-right: $spaceVertical;
  margin-bottom: $spacer-xs;
  margin-top: $spacer-xs; // visually better aligned
  flex-shrink: 0;
  border-radius: 50%;
  text-align: center;
  line-height: 2rem;

  @include phone {
    margin-left: $spacer-sm; // TODO: Suggest to @mmbotelho
  }
}

.c-sendLink {
  border-radius: 0.25rem;
  background-color: $general_2;
  padding: $spacer-sm;
  margin-top: $spacer;

  @include tablet {
    padding: $spacer;
  }

  .link {
    word-break: break-all;
    display: inline; // show the border through multiple lines
  }
}
</style>