<template>
    <div class="pa-4">
        <div v-if="isLocalDeploy">
            <banner-vertical class="ma-3"></banner-vertical>
        </div>
        <a-card title="Members" icon="$fas_users" color="var(--rgbaColor2)">
            <div v-if="isAdmin && !isLocalDeploy" class="email-invite-container">
                <v-combobox
                    v-model="allEmails"
                    :items="[]"
                    multiple
                    chips
                    class="email-input-field"
                    label="Invite your team members"
                    :placeholder="`name@${domainName}`"
                >
                    <template v-slot:selection="{ attrs, item, parent, selected }">
                        <v-chip
                            v-bind="attrs"
                            :class="['chips-base', usernameRules[0](item) ? 'email-valid' : 'email-invalid']"
                            :input-value="selected"
                            label
                            small
                        >
                            <span>
                                {{ item }}
                            </span>
                            <v-icon
                                @click="parent.selectItem(item)"
                                :class="usernameRules[0](item) ? 'email-valid' : 'email-invalid'"
                            >
                                $fas_times
                            </v-icon>
                        </v-chip>
                    </template>
                </v-combobox>
                <v-btn 
                    @click="sendInvitationEmails"
                    :disabled="!allEmails || allEmails.length == 0"
                    dark
                    :style="{
                        'color': !allEmails || allEmails.length == 0 ? 'var(--rgbaColor21) !important' : 'var(--white) !important',
                        'background-color': !allEmails || allEmails.length == 0 ? 'var(--white) !important' : 'var(--themeColor) !important',
                    }"
                    class="invite-btn"
                >
                    Invite
                </v-btn>
            
            </div>
            <div class="team-overview-card">
                <template v-for="user in users">
                    <v-hover 
                        v-slot="{ hover }" 
                        :key="user.email"
                        class="user-details d-flex justify-space-between pa-4"
                    >
                        <div style="position: relative">
                            
                            <owner-name
                                    :owner-name="user.name"
                                    :owner-id="user.id"
                                    :show-name="false"
                                    class="user-details-avatar"
                            />
                            <div class="user-container">
                                <div class="user-details-name">{{user.name}}</div>
                                <div class="user-details-login">{{user.login}}</div>
                            </div>
                            <div class="user-details-type">
                                {{user.role || '-'}}
                            </div>
                            <actions-tray  
                                v-if="hover && isAdmin" 
                                class="table-row-actions" 
                                :actions="actions || []" 
                                :subject=user 
                            />
                        </div>
                    </v-hover>
                </template>
            </div>
        </a-card>
    </div>
</template>

<script>
    import ACard from "@/apps/dashboard/shared/components/ACard"
    import {mapState} from "vuex"
    import OwnerName from "@/apps/dashboard/shared/components/OwnerName"
    import api from "../api"
    import ActionsTray from '@/apps/dashboard/shared/components/ActionsTray'
    import BannerVertical from "../../../shared/components/BannerVertical.vue"

    export default {
        name: "TeamOverview",
        components: {
            ACard,
            OwnerName,
            ActionsTray,
            BannerVertical
        },
        data () {
            let domainName = window.USER_NAME.substr(window.USER_NAME.indexOf("@")+1)
            return {
                showTeamField: false,
                allEmails:[],
                domainName: domainName,
                usernameRules: [
                    (v) => {
                        const pattern = /^(([^<>()[\]\\.,;:\s@"]+(\.[^<>()[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/
                        var ret = pattern.test(v) && v.indexOf("@") != -1 && v.split("@")[1]===domainName
                        this.disableButtons = !ret
                        return ret
                    }
                ],
                actions: [
                    {
                        isValid: item => item.login != window.USER_NAME,
                        icon: item => '$fas_trash',
                        text: item => 'Remove user',
                        func: item => this.removeUser(item),
                        success: (resp, item) => this.removedSuccess(resp, item),
                        failure: (err, item) => this.removedFailure(err, item)
                    }
                ],
                isLocalDeploy: window.DASHBOARD_MODE && window.DASHBOARD_MODE.toLowerCase() == 'local_deploy',
            }
        },
        mounted() {
            this.$store.dispatch('team/getTeamData')
        },
        methods: {
            inviteNewMember(email) {
                let spec = {
                    inviteeName: "there",
                    inviteeEmail: email,
                    websiteHostName: window.location.origin
                }
                api.inviteUsers(spec)
            },
            sendInvitationEmails () {
                let _inviteNewMember = this.inviteNewMember
                let countEmails = 0
                if (this.allEmails && this.allEmails.length > 0) {
                    (this.allEmails || []).forEach(email => {
                        
                        if (this.usernameRules[0](email)) {
                            _inviteNewMember(email)
                            countEmails++;
                        }
                    })
                    let plural = countEmails > 1 ? "s" : ""

                    if (countEmails == 0) {
                        window._AKTO.$emit('SHOW_SNACKBAR', {
                            show: true,
                            text: `Please enter a valid email!`,
                            color: 'red'
                        })

                    } else {

                        window._AKTO.$emit('SHOW_SNACKBAR', {
                            show: true,
                            text: `${countEmails} invitation${plural} sent successfully!`,
                            color: 'green'
                        })
                        this.$store.dispatch('team/getTeamData')
                    }
                    this.allEmails = []
                }
            },
            removeUser (user) {
                return this.$store.dispatch('team/removeUser', user)
            },
            removedSuccess (resp, user) {
                window._AKTO.$emit('SHOW_SNACKBAR', {
                    show: true,
                    text: `${user.login} removed successfully!`,
                    color: 'green'
                })
            },
            removedFailure (err, user) {
                window._AKTO.$emit('SHOW_SNACKBAR', {
                    show: true,
                    text: `There was an error while removing ${user.email}!`,
                    color: 'red'
                })
            },
        },
        computed: {
            ...mapState('team', ['users']),
            isAdmin() {
                return true
            }            
        }
    }
</script>

<style scoped lang="sass">
.team-overview-card
    height: 390px
    overflow-y: auto

.user
    &-add
        cursor: pointer
        margin: auto
        width: 100%
        padding-top: 12px
        display: flex
        justify-content: center

        &-icon
            font-size: 16px !important
            color: var(--teamColor)
            border: 2px solid var(--teamColor)
            border-radius: 50%
            width: 30px !important
            height: 30px !important
            margin-right: 12px

        &-cta
            font-size: 16px
            color: var(--teamColor)
            font-weight: 400
            margin-top: auto
            margin-bottom: auto


    &-container
        width: 65%

    &-details
        &-avatar
            margin-top: auto
            margin-bottom: auto
            width: 15%

        &-name
            font-size: 12px
            color: var(--themeColorDark)
            font-weight: 500

        &-login
            font-size: 12px
            color: var(--teamColor2)
            font-weight: 400

        &-type
            font-size: 12px
            color: var(--teamColor2)
            font-weight: 400
            width: 20%
            margin-top: auto

.dashboard
    &-add
        cursor: pointer
        margin: auto
        width: 100%
        padding-top: 12px
        display: flex
        justify-content: center

        &-icon
            font-size: 16px !important
            color: var(--teamColor)
            border: 2px solid var(--teamColor)
            border-radius: 50%
            width: 30px !important
            height: 30px !important
            margin-right: 12px

        &-cta
            font-size: 16px
            color: var(--teamColor)
            font-weight: 400
            margin-top: auto
            margin-bottom: auto

    &-details
        background-color: var(--themeColorDark17)
        border-radius: 4px
        height: 60px

        &-cards
            font-size: 12px
            color: var(--teamColor2)
            font-weight: 400
            margin-top: auto
            margin-bottom: auto
            padding-right: 16px


        &-name
            font-size: 13px
            color: var(--themeColorDark) !important
            font-weight: 500
            margin-top: auto
            margin-bottom: auto
            padding-left: 16px
.email-valid
  color: var(--v-greenMetric-base)  !important
  font-size: 12px !important

.email-invalid
  color: var(--v-redMetric-base) !important
  font-size: 12px !important

.chips-base
  border-radius: 4px !important
  padding: 8px 0px 8px 8px
  height: 32px
  margin: 8px 8px 4px 0 !important
  background-color: var(--white) !important

.v-select__slot
  border: 1px solid var(--black)

  & .v-label-active
    top: -10px

.invite-btn
    flex: 1
    margin: auto

.email-input-field
    flex: 6
    padding-right: 20px

.email-invite-container
    display: flex
    padding: 16px
    flex-direction: row

.table-row-actions    
    position: absolute
    right: 30px
    padding: 8px 16px !important
    margin: auto

</style>