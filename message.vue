<template>
  <flexbox
    class="message-cell"
    align="stretch">
    <div class="message-cell__hd">
      <i
        :class="typeObj.icon"
      />
    </div>

    <div class="message-cell__bd">

      <div class="message-cell-title">{{ getTitle() }}</div>
      <!-- 9 -->
      <div class="message-cell-des">
        <span>{{ leftContent }}</span>
        <!-- 是工单下的任务、活动 -->
        <span v-if="isWorkOrderTaskOrActivity">
          <span :class="['click-content']" @click="enterDetail">{{ `《${data.title}》` }}</span>
          <span>{{ isWorkOrderTask ? '的任务' : '的活动' }}</span>
          <span :class="['click-content']" @click="enterTaskOrActivityDetail">{{ `《${taskActivityName}》` }}</span>
        </span>
        <!-- 否则 -->
        <span v-else :class="[{ 'is-invalid': isInvalid }, 'click-content']" @click="enterDetail">{{ centerCotent }}</span>
        <span>{{ rightContent }}</span>
      </div>
      <!-- 概要 -->
      <div class="brief">
        <div class="text-one-line">{{ getBrief() }}</div>
        <div
          v-for="(item, index) in contentFields"
          :key="`field-${index}`"
          class="brief-cell">
          <div class="brief-label">{{ item.name }}</div>
          <div v-if="item.formType === 'relative'" class="brief-value">
            <div
              v-for="(rItem, rIndex) in item.value"
              :key="`value-${rIndex}`"
              class="relative-item">
              <span>{{ `${rItem.label}` }}-</span><span @click="relativeClick(rItem)">{{ rItem.name }}</span>
            </div>
          </div>
          <template v-else-if="item.hasDetails && Array.isArray(item.value)">
            <div class="brief-value-wrap">
              <span
                v-for="(subItem,subIndex) in item.value"
                :key="`subValue-${subIndex}`"
              >
                <span
                  class="brief-value is-visit"
                  @click="relativeClick({...subItem,type:'contract'})">{{ subItem.name }}</span>
                <span v-if="subIndex != item.value.length-1" :key="subIndex">、</span>
              </span>
            </div>
          </template>
          <div
            v-else-if="item.hasDetails"
            class="brief-value text-one-line is-visit"
            @click="relativeClick(item)">{{ item.value }}</div>
          <div v-else class="brief-value text-one-line">{{ item.value }}</div>
        </div>
      </div>
    </div>

    <div class="message-cell__ft">
      <div>{{ data.createTime | formatTime }} </div>
      <flexbox justify="end">
        <el-tooltip effect="dark" :disabled="data.isRead == 1" :content="$t('message.message.markAsRead')" placement="left">
          <el-button class="check-btn" :class="{ 'is-unread' :data.isRead == 0 }" @click="messageReadClick">{{ "" }}</el-button>
        </el-tooltip>
      </flexbox>
      <el-button
        class="delete-btn"
        type="link"
        @click="messageDeleteClick">{{ $t('common.handle.delete') }}</el-button>
    </div>
  </flexbox>
</template>

<script>
import { crmActivityDataAPI } from '@/api/crm/common'

import Flexbox from '@/components/Flexbox/Flexbox.vue'

import { isArray, isObject, strToJSON } from '@/utils/types'
import { timeToFormatTime } from '@/utils'
import { separator } from '@/filters/vueNumeralFilter/filters'
import { mapGetters } from 'vuex'
import { getFlowMesLeftContent, getFlowMesTitle } from '@/components/WkWorkFlow/utils/message'

export default {
  // 消息cell
  name: 'MessageCell',
  components: {
    Flexbox

  },
  props: {
    data: Object,
    dataIndex: Number
  },
  data() {
    return {}
  },
  computed: {
    ...mapGetters(['userInfo']),

    // type=192时 解析content
    parseContent() {
      // 具体区分模块
      if (this.data.type === 192) {
        if (!this.data.content) return {}
        const { type, commentedUsers } = JSON.parse(this.data.content)
        let moduleType, moduleName
        const mapData = [
          [[180, 181], ['pm', this.$t('common.navTop.pm')]],
          [[167, 175, 182, 183], ['leads', this.$t('crm.lead.classNames')]],
          [[168, 176, 190, 191, 202, 203, 204], ['customer', this.$t('crm.account.classNames')]],
          [[169, 177, 184, 185], ['contacts', this.$t('crm.contact.classNames')]],
          [[170, 178, 186, 187], ['business', this.$t('crm.opportunity.classNames')]],
          [[171, 179, 188, 189], ['contract', this.$t('common.approval.contracts')]],
          [[198, 199, 200, 201], ['visitPlan', this.$t('crm.visitPlan.visitPlan')]],
          [[224, 225, 226], ['quotation', this.$t('crm.quotation.classNames')]]
        ]
        for (const [kmkey, mval] of mapData) {
          if (kmkey.includes(type)) {
            [moduleType, moduleName] = mval
            break
          }
        }

        return {
          moduleType,
          moduleName,
          commentedUsers,
          contentType: type
        }
      } else if (this.data.type === 197) {
        const contentObj = JSON.parse(this.data.content)
        return contentObj.customer || {}
      }
      return {}
    },

    // 是系统提醒
    // 555: '系统密码规则变更提醒',
    // 556: 'CRM付费版本到期提醒',
    // 558: '系统购买提醒',
    isSystemRemind() {
      return [555, 556, 558].includes(this.data.type)
    },

    // 是否是工单下的任务
    isWorkOrderTask() {
      return [6014, 6015].includes(this.data.type)
    },

    // 是否是工单下的活动
    isWorkOrderActivity() {
      return [6016, 6017].includes(this.data.type)
    },

    // 是工单下的任务、活动
    isWorkOrderTaskOrActivity() {
      return this.isWorkOrderTask || this.isWorkOrderActivity
    },
    // 是否是迭代
    isIterationTask() {
      return [39].includes(this.data.type) && this.data.content && this.data.content.split(',')[2] == 1
    },
    // 任务显示时间的是否为迭代
    isTimeIterationTask() {
      return [1, 2, 3].includes(this.data.type) && this.data.content !== '{}' && JSON.parse(this.data.content).type == 1
    },

    // 工作流type
    isFlowType() {
      return [7000, 7001, 7002, 7003].includes(this.data.type)
    },

    taskActivityContent() {
      if (this.isWorkOrderTaskOrActivity) {
        console.log(this.data.content, '366')
        const content = JSON.parse(this.data.content)
        return content
      }
      return {}
    },

    taskActivityName() {
      if (this.isWorkOrderTaskOrActivity) {
        return this.isWorkOrderTask ? this.taskActivityContent.taskName : this.taskActivityContent.activityName
      }
      return ''
    },

    typeObj() {
      const typesObj = {
        leads: {
          icon: 'wk wk-leads',
          color: '#6995FF',
          type: 'leads'
        },
        customer: {
          icon: 'wk wk-customer',
          color: '#6995FF',
          type: 'customer'
        },
        contacts: {
          icon: 'wk wk-contacts',
          color: '#6995FF',
          type: 'contacts'
        },
        business: {
          icon: 'wk wk-business',
          color: '#6995FF',
          type: 'business'
        },
        contract: {
          icon: 'wk wk-contract',
          color: '#6995FF',
          type: 'contract'
        },
        receivables: {
          icon: 'wk wk-receivables',
          color: '#6995FF',
          type: 'receivables'
        },
        receivablesPlan: {
          icon: 'wk wk-icon-plan-solid',
          color: '#6995FF',
          type: 'receivablesPlan'
        },
        product: {
          icon: 'wk wk-product',
          color: '#6995FF',
          type: 'product'
        },
        stage_examine: {
          icon: 'wk wk-approve',
          color: '#6995FF',
          type: 'stage_examine'
        },
        log: {
          icon: 'wk wk-log',
          color: '#6995FF',
          type: 'log'
        },
        examine: {
          icon: 'wk wk-approve',
          color: '#6995FF',
          type: 'examine'
        },
        task: {
          icon: 'wk wk-o-task',
          color: '#6995FF',
          type: 'task'
        },
        announcement: {
          icon: 'wk wk-announcement',
          color: '#6995FF',
          type: 'announcement'
        },
        schedule: {
          icon: 'wk wk-schedule',
          color: '#6995FF',
          type: 'schedule'
        },
        invoice: {
          icon: 'wk wk-invoice',
          color: '#6995FF',
          type: 'invoice'
        },
        visit: {
          icon: 'wk wk-visit-line',
          color: '#6995FF',
          type: 'visit'
        },
        marketing: {
          icon: 'wk wk-activity-line',
          color: '#6995FF',
          type: 'marketing'
        },
        knowledge: {
          icon: 'wk wk-book',
          color: '#6995FF',
          type: 'knowledge'
        },
        hrm: {
          icon: 'wk wk-employees',
          color: '#6995FF',
          type: 'hrm'
        },
        jxc_purchase: {
          icon: 'wk wk-icon-shop-cart',
          color: '#ffb940',
          type: 'jxc_purchase'
        },
        jxc_retreat: {
          icon: 'wk wk-drafts',
          color: '#fd964a',
          type: 'jxc_retreat'
        },
        jxc_sale: {
          icon: 'wk wk-record',
          color: '#23bbfb',
          type: 'jxc_sale'
        },
        jxc_saleReturn: {
          icon: 'wk wk-approval-13',
          color: '#fd964a',
          type: 'jxc_saleReturn'
        },
        jxc_payment: {
          icon: 'wk wk-icon-category-note',
          color: '#33d08f',
          type: 'jxc_payment'
        },
        jxc_collection: {
          icon: 'wk wk-l-record',
          color: '#00e4bc',
          type: 'jxc_collection'
        },
        jxc_inventory: {
          icon: 'wk wk-icon-search-note',
          color: '#fd5b4a',
          type: 'jxc_inventory'
        },
        jxc_allocation: {
          icon: 'wk wk-approval-11',
          color: '#23bbfb',
          type: 'jxc_allocation'
        },
        jxc_invalid: {
          icon: 'wk wk-approval-11',
          color: '#23bbfb',
          type: 'jxc_invalid'
        },
        pm: {
          icon: 'wk wk-bell',
          color: '#6995FF',
          type: 'pm'
        },
        activity: {
          icon: 'wk wk-icon-message-line',
          color: '#6995FF',
          type: 'activity'
        },
        visitPlan: {
          icon: 'wk wk-icon-task-l',
          color: '#6995FF',
          type: 'visitPlan'
        },
        workorder: {
          icon: 'wk wk-bell',
          color: '#6995FF',
          type: 'workorder'
        },
        offline: {
          icon: 'wk wk-icon-log-l',
          color: '#6995FF',
          type: 'offline'
        },
        outwork: {
          icon: 'wk wk-icon-position2',
          color: '#6995FF',
          type: 'outwork'
        },
        flow: {
          icon: 'wk wk-icon-stage',
          color: '#6995FF',
          type: 'flow'
        },
        quotation: {
          icon: 'wk wk-icon-stage',
          color: '#6995FF',
          type: 'quotation'
        }
      }

      // 1 任务 2 日志 3 oa审批 4公告 5 日程 6 客户管理 8 人资 9 进销存 26 报价单
      // label 20 阶段审批提醒 133 审批通知 134 通过通知 135 拒绝通知  content 内的 type 是CRM对应模块 remarks 是阶段名称
      let key = ''
      if (this.data.label && ((this.data.label <= 5 && this.data.label != -1) || [8, 9].includes(this.data.label))) {
        if (this.data.label <= 5) {
          key = ['task', 'log', 'examine', 'announcement', 'schedule'][this.data.label - 1]
        } else if (this.data.label === 9) {
          if ([53, 54, 55, 150, 3000, 3001, 3002].includes(this.data.type)) {
            key = 'jxc_purchase'
          } else if ([56, 57, 58, 151, 3003, 3004, 3005].includes(this.data.type)) {
            key = 'jxc_retreat'
          } else if ([59, 60, 61, 152, 3006, 3007, 3008].includes(this.data.type)) {
            key = 'jxc_sale'
          } else if ([62, 63, 64, 153, 3009, 3010, 3011].includes(this.data.type)) {
            key = 'jxc_saleReturn'
          } else if ([65, 66, 67, 154, 3012, 3013, 3014].includes(this.data.type)) {
            key = 'jxc_payment'
          } else if ([68, 69, 70, 155, 3015, 3016, 3017].includes(this.data.type)) {
            key = 'jxc_collection'
          } else if ([71, 72, 73, 156, 3018, 3019, 3020].includes(this.data.type)) {
            key = 'jxc_inventory'
          } else if ([74, 75, 76, 157, 3021, 3022, 3023].includes(this.data.type)) {
            key = 'jxc_allocation'
          } else if ([3041, 3042, 3042].includes(this.data.type)) {
            key = 'jxc_invalid'
          }
        } else if (this.data.label === 8) {
          key = 'hrm'
        }
      } else if (this.data.label === 20) {
        key = 'stage_examine' // 阶段审批
      } else if (this.data.label === 22) {
        key = 'pm'
      } else if (this.data.label === 26) { // 报价单 crm
        key = 'quotation'
      } else if (this.data.type === 1017) { // 跟进记录
        key = 'activity'
      } else if ([196, 197, 198, 199, 200, 201, 212].includes(this.data.type)) { // 拜访计划
        key = 'visitPlan'
      } else if (this.isFlowType) { // 工作流
        key = 'flow'
      } else {
        if ([1, 2, 3, 205].includes(this.data.type)) {
          key = 'task'
        } else if ([4, 5, 34, 206].includes(this.data.type)) {
          key = 'log'
        } else if ([6, 7, 25].includes(this.data.type)) {
          key = 'examine'
        } else if ([8].includes(this.data.type)) {
          key = 'announcement'
        } else if ([10, 11, 24, 26, 30, 33, 146, 171, 179, 188, 189, 161, 1000, 1004, 1008, 211].includes(this.data.type)) {
          // 171-跟进记录@  179-评论@  188-评论回复提醒  189-回复提醒
          key = 'contract'
        } else if ([12, 13, 27, 121, 123, 125, 147, 163, 1001, 1005, 1009].includes(this.data.type)) {
          key = 'receivables'
        } else if ([164].includes(this.data.type)) {
          key = 'receivablesPlan'
        } else if ([118, 207].includes(this.data.type)) {
          // 118-线索分配
          key = 'leads'
        } else if ([14, 15, 23, 29, 32, 168, 176, 190, 191, 119, 45, 202, 203, 204, 208].includes(this.data.type)) {
          // 168-跟进记录@  176-评论@  190-评论回复提醒  191-回复提醒 119-客户分配 45-线索转化为客户
          key = 'customer'
        } else if ([16, 17, 120, 122, 124, 168, 169, 177, 184, 185, 209].includes(this.data.type)) {
          // 169-跟进记录@  177-评论@  184-评论回复提醒  185-回复提醒
          key = 'contacts'
        } else if ([18, 19, 167, 175, 182, 183].includes(this.data.type)) {
          // 167-跟进记录@  175-评论@  182-评论回复提醒  183-回复提醒
          key = 'leads'
        } else if ([20, 21].includes(this.data.type)) {
          key = 'product'
        } else if ([22, 28, 31, 170, 178, 186, 187, 210].includes(this.data.type)) {
          // 170-跟进记录@  178-评论@  186-评论回复提醒  187-回复提醒
          key = 'business'
        } else if ([214, 215, 215, 217, 218, 219, 220, 221, 222, 223, 224, 225, 226, 227, 228].includes(this.data.type)) {
          // 227-跟进记录详情评论@  228-报价单详情跟进记录@
          key = 'quotation'
        } else if ([35, 36, 37, 148, 162, 1002, 1006, 1010].includes(this.data.type)) {
          key = 'invoice'
        } else if ([143, 144, 145, 165].includes(this.data.type)) {
          key = 'visit'
        } else if ([166].includes(this.data.type)) {
          key = 'marketing'
        } else if ([41].includes(this.data.type)) {
          key = 'knowledge'
        } else if (this.data.type === 192) {
          key = this.parseContent.moduleType
        } else if ([6000, 6001].includes(this.data.type)) {
          key = 'offline'
        } else if (this.data.type === 999) { // 审批自选
          const { label } = JSON.parse(this.data.content)
          const typeObj = {
            0: 'examine',
            1: 'contract',
            2: 'receivables',
            3: 'invoice',
            4: 'hrm',
            5: 'jxc_purchase',
            6: 'jxc_retreat',
            7: 'jxc_sale',
            8: 'jxc_saleReturn',
            9: 'jxc_payment',
            10: 'jxc_collection',
            11: 'jxc_inventory',
            12: 'jxc_allocation',
            17: 'jxc_invalid',
            20: 'stage_examine',
            21: 'workorder', // 工单
            22: 'quotation' // 报价单
          }
          key = typeObj[label]
        } else if (this.isSystemRemind) {
          key = 'pm'
        } else if (this.isWorkOrderTaskOrActivity || [].includes(this.data.type)) {
          key = 'workorder'
        } else if ([213].includes(this.data.type)) {
          key = 'outwork'
        }
      }
      console.log(typesObj[key])
      return typesObj[key] || {
        icon: 'wk wk-icon-message-line',
        color: '#6995FF',
        type: 'error'
      } // 加入一个默认值增强稳定性，无意义
    },

    leftContent() {
      if (this.isSystemRemind) {
        // 555 密码规则变更
        return this.data.type === 555 ? this.$t('message.message.passwordChangeTips') : this.data.content
      }

      return {        1: `${this.data.realname} Chỉ định thay đổi được giao cho bạn, vui lòng kiểm tra`,
        2: `${this.data.realname} Mời bạn tham gia lần nữa,`,
        3: `${this.data.realname} Chỉ định`,
        4: `${this.data.realname} Đã nhận xét trên nhật ký của bạn,`,
        5: `${this.data.realname} Nhật ký sẽ được gửi cho bạn,`,
        6: `${this.data.realname || this.data.createUserEmail} Nếu bị từ chối`,
        7: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        8: this.$t('message.message.youHaveNewAnnouncement'),
        9: `${this.data.realname} 邀请您参与`,
        10: `${this.data.realname} 拒绝您的`,
        11: `${this.data.realname} 已经审核通过您的`,
        12: `${this.data.realname} 拒绝您的`,
        13: `${this.data.realname} 已经审核通过您的`,
        14: `${this.data.realname} 导入客户数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        15: `${this.data.realname} 取消导入客户数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        16: `${this.data.realname} 导入联系人数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        17: `${this.data.realname} 取消导入联系人数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        18: `${this.data.realname} 导入线索数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        19: `${this.data.realname} 取消导入线索数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        20: `${this.data.realname} 导入产品数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        21: `${this.data.realname} 取消导入产品数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        22: `${this.data.realname} 将您添加为商机`,
        23: `${this.data.realname} 将您添加为客户`,
        24: `${this.data.realname} 将您添加为合同`,
        25: `${this.data.realname} 提交了`,
        26: `${this.data.realname} 提交了`,
        27: `${this.data.realname} 提交了`,
        28: `${this.data.realname} 退出了您商机`,
        29: `${this.data.realname} 退出了您客户`,
        30: `${this.data.realname} 退出了您合同`,
        31: `${this.data.realname} 将您移出了商机`,
        32: `${this.data.realname} 将您移出了客户`,
        33: `${this.data.realname} 将您移出了合同`,
        34: `${this.data.realname} 回复了您评论的日志`,
        35: `${this.data.realname} 拒绝您的`,
        36: `${this.data.realname} 已经审核通过您的`,
        37: `${this.data.realname} 提交了`,
        39: `${this.data.realname}在项目 ${this.data.content && this.data.content.split(',')[0]} 中给 ${this.data.recipientUser === this.userInfo.userId ? '你' : this.data.content && this.data.content.split(',')[1]} 指派了一个${this.data.content && this.data.content.split(',')[2] == 1 ? '迭代' : '任务'} ：`,
        40: `${this.data.realname}在项目 ${this.data.content && this.data.content.split(',')[0]} 中修改了任务：`,
        41: `${this.data.realname} 将文档`,
        45: `您有一个新的线索转化客户`,
        50: `${this.data.realname} 导入员工数据${this.data.title}条，${this.getImportContent(this.data)}`,
        53: `${this.data.realname} 提交了`,
        54: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        55: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        56: `${this.data.realname} 提交了`,
        57: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        58: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        59: `${this.data.realname} 提交了`,
        60: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        61: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        62: `${this.data.realname} 提交了`,
        63: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        64: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        65: `${this.data.realname} 提交了`,
        66: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        67: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        68: `${this.data.realname} 提交了`,
        69: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        70: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        71: `${this.data.realname} 提交了`,
        72: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        73: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        74: `${this.data.realname} 提交了`,
        75: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        76: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        77: `${this.data.realname} 点赞了您的`,
        80: `${this.data.realname} 更新了您`,
        81: `${this.data.realname} 导入定薪数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        82: `${this.data.realname} 导入调薪数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        83: `${this.data.realname} 邀请您填写`,
        84: `${this.data.realname || this.data.createUserEmail} 已经审核通过您的`,
        85: `${this.data.realname || this.data.createUserEmail} 拒绝您的`,
        86: `${this.data.realname} 提交了`,
        // 人资绩效考核
        87: `您有新的绩效目标`,
        88: `您有新的绩效目标`,
        89: `待评定${this.data.realname}的考核`,
        90: '待确认考核结果',
        91: '您的绩效考核',
        92: '您的绩效目标',
        93: `您的绩效目标`,
        94: '您发起的考核',
        95: '您发起的考核',
        96: '您发起的考核',
        97: `${this.data.realname}已为您开通`,
        98: '您的',
        99: `您有新的面试安排，候选人${this.data.title}`,
        101: `您有新的绩效目标`,
        102: `您有新的绩效目标`,
        103: `您有新的绩效目标`,
        104: `您有新的绩效目标`,
        105: `您有新的绩效目标`,
        106: `您有新的绩效目标`,
        107: '您有新的绩效目标',
        108: '您有新的绩效目标',
        118: `${this.data.realname}将线索`,
        119: `${this.data.realname}将客户`,
        120: `${this.data.realname}将您添加为联系人`,
        121: `${this.data.realname}将您添加为回款`,
        122: `${this.data.realname}退出了您联系人`,
        123: `${this.data.realname}退出了您回款`,
        124: `${this.data.realname}将您移出了联系人`,
        125: `${this.data.realname}将您移出了回款`,
        133: `${this.data.realname}提交了`,
        134: `${this.data.realname}通过了`,
        135: `${this.data.realname}拒绝了`,
        136: `${this.data.realname}回复了`,
        137: `${this.data.realname}评论了`,
        138: `${this.data.realname} 将审批`,

        143: `${this.data.realname}将您添加为回访`,
        144: `${this.data.realname}将您移出了回访`,
        145: `${this.data.realname}退出了您回访`,

        146: `${this.data.realname} 将审批`,
        147: `${this.data.realname} 将审批`,
        148: `${this.data.realname} 将审批`,
        149: `${this.data.realname} 将审批`,
        150: `${this.data.realname} 将审批`,
        151: `${this.data.realname} 将审批`,
        152: `${this.data.realname} 将审批`,
        153: `${this.data.realname} 将审批`,
        154: `${this.data.realname} 将审批`,
        155: `${this.data.realname} 将审批`,
        156: `${this.data.realname} 将审批`,
        157: `${this.data.realname} 将审批`,
        158: `${this.data.realname} 将审批`,
        // 阶段审批抄送
        159: `${this.data.realname} 将阶段审批`,
        160: `${this.data.realname} 导入商机数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        161: `${this.data.realname} 导入合同数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        162: `${this.data.realname} 导入发票数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        163: `${this.data.realname} 导入回款数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        164: `${this.data.realname} 导入回款计划数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        165: `${this.data.realname} 导入回访数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        166: `${this.data.realname} 导入市场活动数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        167: `${this.data.realname} 在`,
        168: `${this.data.realname} 在`,
        169: `${this.data.realname} 在`,
        170: `${this.data.realname} 在`,
        171: `${this.data.realname} 在`,
        172: `${this.data.realname} 在`,
        173: `${this.data.realname} 在`,
        175: `${this.data.realname} 在`,
        176: `${this.data.realname} 在`,
        177: `${this.data.realname} 在`,
        178: `${this.data.realname} 在`,
        179: `${this.data.realname} 在`,
        182: `${this.data.realname} 回复了您在`,
        183: `${this.data.realname} 评论了您在`,
        184: `${this.data.realname} 回复了您在`,
        185: `${this.data.realname} 评论了您在`,
        186: `${this.data.realname} 回复了您在`,
        187: `${this.data.realname} 评论了您在`,
        188: `${this.data.realname} 回复了您在`,
        189: `${this.data.realname} 评论了您在`,
        190: `${this.data.realname} 回复了您在`,
        191: `${this.data.realname} 评论了您在`,
        192: `${this.data.realname} 回复了${this.parseContent.commentedUsers}在`,
        196: `${this.data.realname} 导入拜访计划数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        197: '您的拜访计划',
        198: `${this.data.realname} 在`,
        199: `${this.data.realname} 评论了您在`,
        200: `${this.data.realname} 回复了您在`,
        201: `${this.data.realname} 在`,
        202: `${this.data.realname} 在`,
        203: `${this.data.realname} 评论了您在`,
        204: `${this.data.realname} 回复了您在`,
        // 客户跟进记录 跟进回复
        205: `${this.data.realname} 评论了您`,
        206: `${this.data.realname} 评论了您`,
        207: `${this.data.realname} 评论了您`,
        208: `${this.data.realname} 评论了您`,
        209: `${this.data.realname} 评论了您`,
        210: `${this.data.realname} 评论了您`,
        211: `${this.data.realname} 评论了您`,
        212: `${this.data.realname} 评论了您`,
        213: `${this.data.realname} 评论了您`,
        // 报价单 团队
        214: `${this.data.realname} 将您添加为报价单`,
        215: `${this.data.realname} 退出了您报价单`,
        216: `${this.data.realname} 将您移除报价单`,
        // 报价单 审批
        217: `${this.data.realname} 提交了`,
        218: `${this.data.realname} 已经审核通过您的`,
        219: `${this.data.realname} 拒绝您的`,
        220: `${this.data.realname} 恢复了您审批的`,
        221: ``,
        222: ``,
        // 报价单 导入
        223: `${this.data.realname} 导入报价单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        // 报价单提醒
        224: `${this.data.realname} 在`,
        225: `${this.data.realname} 评论了您`,
        226: `${this.data.realname} 评论了您`,
        227: `${this.data.realname} 在`,
        228: `${this.data.realname} 在`,

        // 审批自选通知
        999: `您提交的`,
        1000: `${this.data.realname} 恢复了您的`,
        1001: `${this.data.realname} 恢复了您的`,
        1002: `${this.data.realname} 恢复了您的`,
        1003: `${this.data.realname} 恢复了您的`,
        1004: `${this.data.realname} 拒绝了您审批的`,
        1005: `${this.data.realname} 拒绝了您审批的`,
        1006: `${this.data.realname} 拒绝了您审批的`,
        1007: `${this.data.realname} 拒绝了您审批的`,
        1008: `${this.data.realname} 恢复了您审批的`,
        1009: `${this.data.realname} 恢复了您审批的`,
        1010: `${this.data.realname} 恢复了您审批的`,
        1011: `${this.data.realname} 恢复了您审批的`,
        1016: `${this.data.realname} 导入跟进记录数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        1017: `${this.data.realname} 点赞了您的`,
        // 导入邮件联系人通知
        1500: `${this.data.realname} 导入邮件联系人数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        // 导入短信联系人通知
        1501: `${this.data.realname} 导入短信联系人数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        2000: `${this.data.realname} 恢复了您的`,
        2001: `${this.data.realname} 拒绝了您审批的`,
        2002: `${this.data.realname} 恢复了您审批的`,
        3000: `${this.data.realname} 恢复了您的`,
        3001: `${this.data.realname} 拒绝了您审批的`,
        3002: `${this.data.realname} 恢复了您审批的`,
        3003: `${this.data.realname} 恢复了您的`,
        3004: `${this.data.realname} 拒绝了您审批的`,
        3005: `${this.data.realname} 恢复了您审批的`,
        3006: `${this.data.realname} 恢复了您的`,
        3007: `${this.data.realname} 拒绝了您审批的`,
        3008: `${this.data.realname} 恢复了您审批的`,
        3009: `${this.data.realname} 恢复了您的`,
        3010: `${this.data.realname} 拒绝了您审批的`,
        3011: `${this.data.realname} 恢复了您审批的`,
        3012: `${this.data.realname} 恢复了您的`,
        3013: `${this.data.realname} 拒绝了您审批的`,
        3014: `${this.data.realname} 恢复了您审批的`,
        3015: `${this.data.realname} 恢复了您的`,
        3016: `${this.data.realname} 拒绝了您审批的`,
        3017: `${this.data.realname} 恢复了您审批的`,
        3018: `${this.data.realname} 恢复了您的`,
        3019: `${this.data.realname} 拒绝了您审批的`,
        3020: `${this.data.realname} 恢复了您审批的`,
        3021: `${this.data.realname} 恢复了您的`,
        3022: `${this.data.realname} 拒绝了您审批的`,
        3023: `${this.data.realname} 恢复了您审批的`,
        4000: `${this.data.realname} 恢复了您的`,
        4001: `${this.data.realname} 拒绝了您审批的`,
        4002: `${this.data.realname} 恢复了您审批的`,
        // 进销存
        3032: '',
        3033: '',
        3034: '',
        3035: '',
        3036: '',
        3037: '',
        3038: '',
        3039: '',
        3040: '待审批提醒',
        3041: '审批拒绝提醒',
        3042: '审批通过提醒',
        3043: '',
        3044: '',
        3045: '',
        3046: `${this.data.realname} 导入产品数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3047: `${this.data.realname} 取消导入产品数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3048: `${this.data.realname} 导入供应商数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3049: `${this.data.realname} 取消导入供应商数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3050: `${this.data.realname} 导入采购订单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3051: `${this.data.realname} 取消导入采购订单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3052: `${this.data.realname} 导入采购退货单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3053: `${this.data.realname} 取消导入采购退货单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3054: `${this.data.realname} 导入销售订单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3055: `${this.data.realname} 取消导入销售订单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3056: `${this.data.realname} 导入销售退货单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3057: `${this.data.realname} 取消导入销售退货单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3058: `${this.data.realname} 导入入库单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3059: `${this.data.realname} 取消导入入库单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3060: `${this.data.realname} 导入出库单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3061: `${this.data.realname} 取消导入出库单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3062: `${this.data.realname} 导入付款单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3063: `${this.data.realname} 取消导入付款单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3064: `${this.data.realname} 导入回款单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3065: `${this.data.realname} 取消导入回款单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3066: `${this.data.realname} 导入盘点数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3067: `${this.data.realname} 取消导入盘点数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3068: `${this.data.realname} 导入调拨数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3069: `${this.data.realname} 取消导入调拨数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3070: `${this.data.realname} 导入仓库数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3071: `${this.data.realname} 取消导入仓库数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3072: `${this.data.realname} 导入产品报废数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3073: `${this.data.realname} 取消导入产品报废数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3074: `${this.data.realname} 导入备件图册数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        3075: `${this.data.realname} 取消导入备件图册数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        // 工单
        6000: `${this.data.realname} 导入线下工单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        6001: `${this.data.realname} 取消导入线下工单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        6003: `${this.data.realname} 将`,
        6004: '待审批提醒',
        6005: `${this.data.realname} 拒绝了您审批的`,
        6006: '审批通过提醒',
        6007: `${this.data.realname} 恢复了您的`,
        6008: `${this.data.realname} 拒绝了您审批的`,
        6009: `${this.data.realname} 恢复了您审批的`,
        6011: `${this.data.realname} 在`,
        6012: `${this.data.realname} 在`,
        6013: `${this.data.realname} 将审批`,
        6014: `${this.data.realname} 邀请您参加`,
        6015: `${this.data.realname} 将`,
        6016: `${this.data.realname} 邀请您参加`,
        6017: `${this.data.realname} 将`,
        6018: `${this.data.realname} 导入工单数据${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        6019: `${this.data.realname} 取消导入工单数据，已导入${this.data.title || 0}条，${this.getImportContent(this.data)}`,
        6020: `${this.data.realname} 将审批`
      }[this.data.type] || this.getLeftContent()
    },

    centerCotent() {
      // 导入提示与其他不一样
      if (this.isImportType) {
        // title 是总数 content 是错误数据 valid 错误文件是否有效 1 有效 0 失效
        const list = this.data.content ? this.data.content.split(',') : []
        const errSize = Number(list[0] || 0)
        if (errSize > 0) {
          return this.data.valid === 0 ? '已失效' : '点击下载错误数据'
        }
        return ''
      } else {
        if (this.data.type === 80) {
          const datas = this.data.content ? this.data.content.split(',') : []
          if (datas && datas.length > 1) {
            return `《${datas[0]}年${datas[1]}月的工资条》`
          }
        } else if (this.data.type === 83) {
          return '《我的档案》'
        } else if (this.data.type === 97) {
          return '人力资源'
        } else if (this.data.type === 99) {
          return ''
        } else if (this.data.type === 999) {
          const { label } = JSON.parse(this.data.content)
          if (label === 20) {
            return '《阶段流程》'
          }
        } else if (this.data.type === 197) {
          const contentObj = JSON.parse(this.data.content) || {}
          const dataObj = contentObj.customer || {}
          return `《${dataObj.visitPlanName || ''}》`
        } else if (this.isSystemRemind) {
          // 555 密码规则变更
          return this.data.type == 555 ? '去修改>>' : ''
        } else if (this.isFlowType) {
          if ([7000, 7002].includes(this.data.type)) {
            return '立即处理'
          }
          return '立即查看'
        }
        return `《${this.data.title || ''}》`
      }
    },

    isInvalid() {
      if (this.isIterationTask) {
        return true
      } else if (this.isTimeIterationTask) {
        return true
      } else if (this.isImportType) {
        // title 是总数 content 是错误数据 valid 错误文件是否有效 1 有效 0 失效
        return this.data.valid == 0
      } else {
        return false
      }
    },

    /**
     * 是导入type
     * 50 人资导入类型
     */
    isImportType() {
      return (this.data.type >= 14 && this.data.type <= 21) ||
        [50, 81, 82, 1016, 196, 223, 1500, 1501, 6000, 6018, 6019].includes(this.data.type) ||
        (this.data.type >= 160 && this.data.type <= 166) ||
        (this.data.type >= 3046 && this.data.type <= 3075)
    },

    rightContent() {
      return {

        1: `${this.isTimeIterationTask ? ' :  Nhiệm vụ đã được giao cho bạn,' : ' vui lòng kiểm tra ngay'}`,
        2: `${this.isTimeIterationTask ? 'vui lòng kiểm tra:' : 'Nhiệm vụ, vui lòng kiểm tra ngay'}`,
        3: `${this.isTimeIterationTask ? 'đánh dấu kết thúc lặp lại' : 'đánh dấu kết thúc nhiệm vụ'}`,
        4: 'vui lòng kiểm tra',
        5: 'vui lòng kiểm tra',
        6: '，vui lòng xử lý ngay',
        7: '，vui lòng kiểm tra',
        8: '，vui lòng kiểm tra',
        9: '的日程，请及时查看',
        10: '合同，请及时处理',
        11: '合同，请及时查看',
        12: '回款，请及时处理',
        13: '回款，请及时查看',
        14: '',
        15: '',
        16: '',
        17: '',
        18: '',
        19: '',
        20: '',
        21: '',
        22: '的成员',
        23: '的成员',
        24: '的成员',
        25: '，请及时处理',
        26: '合同审批，请及时处理',
        27: '回款审批，请及时处理',
        28: '的团队',
        29: '的团队',
        30: '的团队',
        31: '的团队',
        32: '的团队',
        33: '的团队',
        34: `，请及时查看`,
        35: '发票，请及时处理',
        36: '发票审批，请及时查看',
        37: '发票审批，请及时处理',
        40: `的状态为 ${this.data.content && this.data.content.split(',')[1]}`,
        41: '分享给您，请及时查看',
        45: '，请及时查看',
        53: '采购订单审批，请及时处理',
        54: `采购订单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        55: '采购订单，请及时查看',
        56: '采购退货单审批，请及时处理',
        57: `采购退货单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        58: '采购退货单，请及时查看',
        59: '销售订单审批，请及时处理',
        60: `销售订单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        61: '销售订单，请及时查看',
        62: '销售退货单审批，请及时处理',
        63: `销售退货单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        64: '销售退货单，请及时查看',
        65: '付款单审批，请及时处理',
        66: `付款单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        67: '付款单，请及时查看',
        68: '回款单审批，请及时处理',
        69: `回款单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        70: '回款单，请及时查看',
        71: '盘点审批，请及时处理',
        72: `盘点审批，拒绝理由：“${this.data.content}”，请及时处理`,
        73: '盘点，请及时查看',
        74: '调拨审批，请及时处理',
        75: `调拨审批，拒绝理由：“${this.data.content}”，请及时处理`,
        76: '调拨，请及时查看',
        77: '日志，请及时查看',
        80: `，有问题请及时与${this.data.realname}反馈`,
        81: '',
        82: '',
        83: '，请及时处理',
        84: '薪资审批，请及时查看',
        85: `薪资审批，拒绝理由：“${this.data.content}”，请及时处理`,
        86: '薪资审批，请及时处理',
        // 人资绩效考核
        87: '指标待填写，请及时处理',
        88: '指标待确认，请及时处理',
        89: '，请及时处理',
        90: '，请及时处理',
        91: '已完成，请及时查看',
        92: `被${this.data.realname}驳回，请重新填写`,
        93: `被${this.data.realname}驳回，请重新评定`,
        94: '已全部完成填写，请及时开启评定',
        95: '已全部完成评定，请及时开启结果确认',
        96: '已完成结果确认，请及时归档',
        97: '员工端，请及时查看',
        98: '已生成，请及时查看',
        99: '，请及时查看',
        101: '，待评分，请及时处理',
        102: '，已评分，请及时查看',
        103: `，被${this.data.realname}驳回，请重新填写`,
        104: '，结果已审核，请及时查看',
        105: '，结果待审核，请及时查看',
        106: '结果申诉待处理，请及时查看',
        107: '结果申诉通过，请及时查看',
        108: '结果申诉驳回，请及时查看',
        118: '分配给您，请及时处理',
        119: '分配给您，请及时处理',
        120: '的成员',
        121: '的成员',
        122: '的团队',
        123: '的团队',
        124: '的团队',
        125: '的团队',
        132: '导出成功',
        136: '任务，请及时查看',
        137: '任务，请及时查看',
        138: '抄送给您，请及时查看',

        143: `的成员`,
        144: `的团队`,
        145: `的团队`,

        146: '抄送给您，请及时查看',
        147: '抄送给您，请及时查看',
        148: '抄送给您，请及时查看',
        149: '抄送给您，请及时查看',
        150: '抄送给您，请及时查看',
        151: '抄送给您，请及时查看',
        152: '抄送给您，请及时查看',
        153: '抄送给您，请及时查看',
        154: '抄送给您，请及时查看',
        155: '抄送给您，请及时查看',
        156: '抄送给您，请及时查看',
        157: '抄送给您，请及时查看',
        158: '抄送给您，请及时查看',
        // 阶段审批抄送
        159: '抄送给您，请及时查看',
        160: '',
        161: '',
        162: '',
        163: '',
        164: '',
        165: '',
        166: '',
        167: '线索中@了您，请及时查看',
        168: '客户中@了您，请及时查看',
        169: '联系人中@了您，请及时查看',
        170: '商机中@了您，请及时查看',
        171: '合同中@了您，请及时查看',
        172: '任务中@了您，请及时查看',
        173: '日志中@了您，请及时查看',
        175: '线索的跟进记录中@了您，请及时查看',
        176: '客户的跟进记录中@了您，请及时查看',
        177: '联系人的跟进记录中@了您，请及时查看',
        178: '商机的跟进记录中@了您，请及时查看',
        179: '合同的跟进记录中@了您，请及时查看',
        182: `线索的跟进记录评论，请及时查看`,
        183: '线索的跟进记录，请及时查看',
        184: `联系人的跟进记录评论，请及时查看`,
        185: '联系人的跟进记录，请及时查看',
        186: `商机的跟进记录评论，请及时查看`,
        187: '商机的跟进记录，请及时查看',
        188: `合同的跟进记录评论，请及时查看`,
        189: '合同的跟进记录，请及时查看',
        190: `客户的跟进记录评论，请及时查看`,
        191: '客户的跟进记录，请及时查看',
        196: '',
        197: `将在《${this.parseContent.visitTime || '--'}》开始，请及时处理`,
        198: '拜访计划的跟进记录中@了您，请及时查看',
        199: '拜访计划的跟进记录，请及时查看',
        200: `拜访计划的跟进记录评论，请及时查看`,
        201: '拜访计划中@了您，请及时查看',
        202: '客户的外勤签到中@了您，请及时查看',
        203: '客户的外勤签到，请及时查看',
        204: '客户的外勤签到评论，请及时查看',
        // 客户跟进记录 跟进回复
        205: '任务的跟进记录，并且需要您回复，请及时处理',
        206: '日志的跟进记录，并且需要您回复，请及时处理',
        207: '线索的跟进记录，并且需要您回复，请及时处理',
        208: '客户的跟进记录，并且需要您回复，请及时处理',
        209: '联系人的跟进记录，并且需要您回复，请及时处理',
        210: '商机的跟进记录，并且需要您回复，请及时处理',
        211: '合同的跟进记录，并且需要您回复，请及时处理',
        212: '拜访计划的跟进记录，并且需要您回复，请及时处理',
        213: '外勤签到的跟进记录，并且需要您回复，请及时处理',
        214: '的成员',
        215: '的团队',
        216: '的团队',
        217: `报价单审批，请及时处理`,
        218: `报价单，请及时查看`,
        219: '报价单，请及时处理',
        220: `报价单，请及时处理`,
        221: `报价单节点拒绝通知`,
        222: `报价单节点恢复通知`,
        223: ``,
        224: `报价单评论回复提醒`,
        225: `报价单的跟进记录，并且需要您回复，请及时处理`,
        226: `报价单的跟进记录，请及时查看`,
        227: '报价单的跟进记录中@了您，请及时查看',
        228: '报价单中@了您，请及时查看',
        // 审批自选通知
        999: '审批需要您配置审批流程处理人，请及时处理',
        1000: '合同，请及时处理',
        1001: '回款，请及时处理',
        1002: '发票，请及时处理',
        1003: '阶段，请及时处理',
        1004: '合同，请及时处理',
        1005: '回款，请及时处理',
        1006: '发票，请及时处理',
        1007: '阶段，请及时处理',
        1008: '合同，请及时处理',
        1009: '回款，请及时处理',
        1010: '发票，请及时处理',
        1011: '阶段，请及时处理',
        1016: '',
        1017: '跟进记录，请及时查看',
        1500: '',
        1501: '',
        2000: '，请及时处理',
        2001: '，请及时处理',
        2002: '，请及时处理',
        3000: '采购订单，请及时处理',
        3001: '采购订单，请及时处理',
        3002: '采购订单，请及时处理',
        3003: '采购退货单，请及时处理',
        3004: '采购退货单，请及时处理',
        3005: '采购退货单，请及时处理',
        3006: '销售订单，请及时处理',
        3007: '销售订单，请及时处理',
        3008: '销售订单，请及时处理',
        3009: '销售退货单，请及时处理',
        3010: '销售退货单，请及时处理',
        3011: '销售退货单，请及时处理',
        3012: '付款单，请及时处理',
        3013: '付款单，请及时处理',
        3014: '付款单，请及时处理',
        3015: '回款单，请及时处理',
        3016: '回款单，请及时处理',
        3017: '回款单，请及时处理',
        3018: '库存盘点，请及时处理',
        3019: '库存盘点，请及时处理',
        3020: '库存盘点，请及时处理',
        3021: '库存调拨，请及时处理',
        3022: '库存调拨，请及时处理',
        3023: '库存调拨，请及时处理',
        4000: '薪资审批，请及时处理',
        4001: '薪资审批，请及时处理',
        4002: '薪资审批，请及时处理',
        // 进销存
        3040: '产品报废单审批，请及时处理',
        3041: `产品报废单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        3042: '产品报废单，请及时查看',
        // 工单
        6000: '',
        6001: '',
        6003: '工单分配给您，请及时查看',

        6004: '工单审批，请及时处理',
        6005: `工单审批，拒绝理由：“${this.data.content}”，请及时处理`,
        6006: '工单，请及时查看',
        6007: '审批恢复提醒',
        6008: '工单，请及时处理',
        6009: '工单，请及时处理',
        6011: '线下工单的跟进记录中@了您，请及时查看',
        6012: '工单的跟进记录中@了您，请及时查看',
        6013: '抄送给您，请及时查看',
        6014: '，请及时查看',
        6015: '标记结束',
        6016: '，请及时查看',
        6017: '标记结束',
        6018: '',
        6019: '',
        6020: '抄送给您，请及时查看'
      }[this.data.type] || this.getRightContent(this.data.type)
    },

    // 简介字段
    contentFields() {
      return this.getContentFields()
    }

    // // 展示文件简介
    // showBrief() {
    //   // 日志 4 5 77
    //   return [4, 5, 77].includes(this.data.type)
    // }
  },
  watch: {},
  mounted() {},

  beforeDestroy() {},
  methods: {
    async enterDetail() {
      console.log(this.data.type, 'this.data.type')
      if (this.isInvalid) {
        return
      }

      // 是导入提醒
      if (this.isImportType) {
        this.$emit('download', this.data.messageId, this.dataIndex)
      } else {
        const type = this.typeObj.type // 默认type
        // if (type === 'stage_examine') {
        //   // 根据content 内容的type 校准为 CRM模块关键词
        //   const contentObj = JSON.parse(this.data.content) || {}
        //   type = crmTypeModel.typeToKeyData[contentObj.type]
        // }
        // 特殊CRM类型，需要讲typeId换成模块id 只针对跟进记录相关
        if ([
          // 线索
          175, 182, 183,
          // 客户
          176, 190, 191, 202, 203, 204,
          // 联系人
          177, 184, 185,
          // 商机
          178, 186, 187,
          // 合同
          179, 188, 189,
          // 需要具体区分content的type对应的模块
          192,
          // 拜访计划
          198, 199, 200, 201,
          // 跟进回复提醒
          205, 206, 207, 208, 209, 210, 211, 212, 213,
          // 报价单
          225, 226, 227, 228
        ].includes(this.data.type)) {
          const resData = await crmActivityDataAPI(this.data.typeId)
          if (resData.data) {
            this.data.typeId = resData.data.activityTypeId
          }
        }

        // 审批自选
        if (this.data.type == 999) {
          const data = JSON.parse(this.data.content)
          const { typeId } = data
          this.data.typeId = typeId
        }

        if (![1500, 1501].includes(this.data.type)) {
          this.$emit('detail', type, this.data.typeId, this.dataIndex, this.data)
        }

        // 未读触发读
        if (this.data.isRead == 0) {
          this.$emit('read', this.data.messageId, this.dataIndex)
        }
      }
    },

    /**
     * @description: 进入任务活动详情
     * @return {*}
     */
    enterTaskOrActivityDetail() {
      console.log(JSON.stringify(this.taskActivityContent), 'this.taskActivityContent')
      const typeId = this.taskActivityContent.taskId || this.taskActivityContent.activityId
      const type = this.isWorkOrderTask ? 'workOrderTask' : 'workOrderActivity'
      this.$emit('detail', type, typeId, this.dataIndex, this.taskActivityContent)

      // 未读触发读
      if (this.data.isRead == 0) {
        this.$emit('read', this.data.messageId, this.dataIndex)
      }
    },

    /**
     * @description: 相关详情查看
     * @param {*}
     * @return {*}
     */
    relativeClick(item) {
      this.$emit('detail', item.type, item.id, this.dataIndex, this.data)
    },

    /**
     * 标记已读
     */
    messageReadClick() {
      if (this.data.isRead == 0) {
        this.$emit('read', this.data.messageId, this.dataIndex)
      }
    },

    /**
     * 消息删除
     */
    messageDeleteClick() {
      this.$emit('delete', this.data.messageId, this.dataIndex)
    },

    getImportContent({ title, content }) {
      if (!this.getImportContent) {
        return
      }
      const list = content ? content.split(',') : []
      const totalSize = Number(title || '0')
      const updateSize = Number(list[1] || '0')
      const errSize = Number(list[0] || '0')
      return `覆盖${updateSize}条，导入成功${totalSize - errSize}条，导入失败${errSize}条。`
    },

    /**
     * @description: 获取标题
     * @param {*}
     * @return {*}
     */
    getTitle() {
      return {

        1: `${this.isTimeIterationTask ? '迭代分配提醒' : '任务分配提醒'}`,
        2: `${this.isTimeIterationTask ? '迭代邀请提醒' : '任务邀请提醒'}`,
        3: `${this.isTimeIterationTask ? '迭代完成提醒' : '任务完成提醒'}`,
        4: '日志评论提醒',
        5: '日志发送提醒',
        6: '审批拒绝提醒',
        7: '审批通过提醒',
        8: '公告提醒',
        9: '日程提醒',
        10: '审批拒绝提醒',
        11: '审批通过提醒',
        12: '审批拒绝提醒',
        13: '审批通过提醒',
        14: '导入提醒',
        15: '取消导入提醒',
        16: '导入提醒',
        17: '取消导入提醒',
        18: '导入提醒',
        19: '取消导入提醒',
        20: '导入提醒',
        21: '取消导入提醒',
        22: '添加团队成员提醒',
        23: '添加团队成员提醒',
        24: '添加团队成员提醒',
        25: '待审批提醒',
        26: '待审批提醒',
        27: '待审批提醒',
        28: '退出团队成员提醒',
        29: '退出团队成员提醒',
        30: '退出团队成员提醒',
        31: '移出团队提醒',
        32: '移出团队提醒',
        33: '移出团队提醒',
        34: '日志评论提醒',
        35: '审批拒绝提醒',
        36: '审批通过提醒',
        37: '待审批提醒',
        39: '项目管理通知',
        40: '项目管理通知',
        41: '文档分享提醒',
        45: '线索转化为客户提醒',
        50: '导入提醒',
        53: '待审批提醒',
        54: '审批拒绝提醒',
        55: '审批通过提醒',
        56: '待审批提醒',
        57: '审批拒绝提醒',
        58: '审批通过提醒',
        59: '待审批提醒',
        60: '审批拒绝提醒',
        61: '审批通过提醒',
        62: '待审批提醒',
        63: '审批拒绝提醒',
        64: '审批通过提醒',
        65: '待审批提醒',
        66: '审批拒绝提醒',
        67: '审批通过提醒',
        68: '待审批提醒',
        69: '审批拒绝提醒',
        70: '审批通过提醒',
        71: '待审批提醒',
        72: '审批拒绝提醒',
        73: '审批通过提醒',
        74: '待审批提醒',
        75: '审批拒绝提醒',
        76: '审批通过提醒',
        77: '日志点赞提醒',
        80: '个人档案更新提醒',
        81: '导入提醒',
        82: '导入提醒',
        83: '个人档案填写提醒',
        84: '审批通过提醒',
        85: '审批拒绝提醒',
        86: '待审批提醒',
        // 人资绩效考核
        87: '考核目标待填写提醒',
        88: '考核目标待确认提醒',
        89: '考核目标待评定提醒',
        90: '考核结果待确认提醒',
        91: '考核完成提醒',
        92: '考核目标被驳回提醒',
        93: '考核评定被驳回提醒',
        94: '考核评定待开启提醒',
        95: '考核结果确认待开启提醒',
        96: '考核待归档提醒',
        97: '员工端开通提醒',
        98: '社保核算提醒', // 员工社保提醒
        99: '候选人面试提醒',
        101: '考核目标待评分提醒',
        102: '考核目标已评分提醒',
        103: '考核目标评分被驳回提醒',
        104: '考核目标结果已审核提醒',
        105: '考核目标结果待审核提醒',
        106: '考核目标结果申诉待处理提醒',
        107: '考核目标结果申诉通过提醒',
        108: '考核目标结果申诉驳回提醒',
        118: '线索分配提醒',
        119: '客户分配提醒',
        120: '添加团队成员提醒',
        121: '添加团队成员提醒',
        122: '退出团队成员提醒',
        123: '退出团队成员提醒',
        124: '移出团队提醒',
        125: '移出团队提醒',
        133: '待审批提醒',
        134: '审批通过提醒',
        135: '审批拒绝提醒',
        143: '添加团队成员提醒',
        144: '移出团队提醒',
        145: '退出团队成员提醒',
        146: '审批抄送提醒',
        147: '审批抄送提醒',
        148: '审批抄送提醒',
        149: '审批抄送提醒',
        150: '审批抄送提醒',
        151: '审批抄送提醒',
        152: '审批抄送提醒',
        153: '审批抄送提醒',
        154: '审批抄送提醒',
        155: '审批抄送提醒',
        156: '审批抄送提醒',
        157: '审批抄送提醒',
        158: '审批抄送提醒',
        // 阶段审批抄送
        159: '阶段审批抄送提醒',
        161: '导入提醒',
        162: '导入提醒',
        163: '导入提醒',
        164: '导入提醒',
        165: '导入提醒',
        166: '导入提醒',
        196: '导入提醒',
        197: '拜访计划提醒',
        // 报价单
        217: '待审批提醒',
        218: '审批通过提醒',
        219: '审批拒绝提醒',
        220: '审批恢复提醒',
        221: '报价单节点拒绝通知',
        222: '报价单节点恢复通知',
        223: '导入提醒',

        // 密码规则变更
        555: '系统密码规则变更提醒',
        556: 'CRM付费版本到期提醒',
        558: '系统购买提醒',
        // 审批自选通知
        999: '待处理流程提醒',
        1000: '审批恢复提醒',
        1001: '审批恢复提醒',
        1002: '审批恢复提醒',
        1003: '审批恢复提醒',
        1004: '审批拒绝提醒',
        1005: '审批拒绝提醒',
        1006: '审批拒绝提醒',
        1007: '审批拒绝提醒',
        1008: '审批恢复提醒',
        1009: '审批恢复提醒',
        1010: '审批恢复提醒',
        1011: '审批恢复提醒',
        1016: '导入提醒',
        1017: '跟进记录点赞提醒',
        1500: '导入提醒',
        1501: '导入提醒',
        2000: '审批恢复提醒',
        2001: '审批恢复提醒',
        3003: '审批恢复提醒',
        3004: '审批拒绝提醒',
        3005: '审批恢复提醒',
        3006: '审批恢复提醒',
        3007: '审批拒绝提醒',
        3008: '审批恢复提醒',
        3009: '审批恢复提醒',
        3010: '审批拒绝提醒',
        3011: '审批恢复提醒',
        3012: '审批恢复提醒',
        3013: '审批拒绝提醒',
        3014: '审批恢复提醒',
        3015: '审批恢复提醒',
        3016: '审批拒绝提醒',
        3017: '审批恢复提醒',
        3018: '审批恢复提醒',
        3019: '审批拒绝提醒',
        3020: '审批恢复提醒',
        3021: '审批恢复提醒',
        3022: '审批拒绝提醒',
        3023: '审批恢复提醒',
        // 进销存
        3032: '产品报废单恢复通知',
        3033: '产品报废单节点拒绝通知',
        3034: '产品报废单节点恢复通知',
        3035: '产品报废单结束通知',
        3036: '备件图册恢复通知',
        3037: '备件图册节点拒绝通知',
        3038: '备件图册节点恢复通知',
        3039: '备件图册结束通知',
        3040: `${this.data.realname}提交了`,
        3041: `${this.data.realname}拒绝了`,
        3042: `${this.data.realname}通过了`,
        3043: '备件图册待审核审批提醒',
        3044: '备件图册拒绝通知',
        3045: '备件图册通过通知',
        3046: '导入提醒',
        3047: '取消导入提醒',
        3048: '导入提醒',
        3049: '取消导入提醒',
        3050: '导入提醒',
        3051: '取消导入提醒',
        3052: '导入提醒',
        3053: '取消导入提醒',
        3054: '导入提醒',
        3055: '取消导入提醒',
        3056: '导入提醒',
        3057: '取消导入提醒',
        3058: '导入提醒',
        3059: '取消导入提醒',
        3060: '导入提醒',
        3061: '取消导入提醒',
        3062: '导入提醒',
        3063: '取消导入提醒',
        3064: '导入提醒',
        3065: '取消导入提醒',
        3066: '导入提醒',
        3067: '取消导入提醒',
        3068: '导入提醒',
        3069: '取消导入提醒',
        3070: '导入提醒',
        3071: '取消导入提醒',
        3072: '导入提醒',
        3073: '取消导入提醒',
        3074: '导入提醒',
        3075: '取消导入提醒',
        // 工单
        6000: '导入提醒',
        6001: '取消导入提醒',
        6003: '工单分配提醒',
        6004: '待审批提醒',
        6005: '审批拒绝提醒',
        6006: '审批通过提醒',
        6007: '审批恢复提醒',
        6008: '审批拒绝提醒',
        6009: '审批恢复提醒',
        6013: '审批抄送提醒',
        6014: '任务邀请提醒',
        6015: '任务完成提醒',
        6016: '活动邀请提醒',
        6017: '活动完成提醒'
      }[this.data.type] || this.getCustomTitle()
    },

    /**
     * @description: 获取特殊的标题
     * @return {*}
     */
    getCustomTitle() {
      // 工作流提醒
      if (this.isFlowType) {
        return getFlowMesTitle(this.data)
      }
      return ''
    },

    /**
     * @description: 获取特殊的左侧内容
     * @return {*}
     */
    getLeftContent() {
      if (this.isFlowType) {
        return getFlowMesLeftContent(this.data, this.userInfo)
      }
      return ''
    },

    /**
     * @description: 获取特殊的右侧内容
     * @return {*}
     */
    getRightContent() {
      const { content, type } = this.data
      if ([133, 134, 135].includes(type)) {
        const contentObj = JSON.parse(content) || {}
        const crmTypeModel = wkGlobal.get('crm.utils.crmTypeModel')
        const typeName = crmTypeModel.typeToNameData[contentObj.type]
        return {
          133: `${typeName}阶段审批，请及时处理`,
          134: `${typeName}阶段审批，请及时查看`,
          135: `${typeName}阶段审批，请及时处理`
        }[type]
      } else if ([192].includes(type)) {
        if ([202, 203, 204].includes(Number(this.parseContent.contentType))) {
          return `${this.parseContent.moduleName}的外勤签到评论，请及时查看`
        }
        return `${this.parseContent.moduleName}的跟进记录评论，请及时查看`
      }
      return ''
    },

    /**
     * @description: 获取概要
     * @param {*}
     * @return {*}
     */
    getBrief() {
      // const { type } = this.data
      // 136 137 任务评论 4 日志评论
      // if ([4].includes(type)) {
      //   return `评论内容 ${this.data.content || ''}`
      // }
      // else if (type == 5) {
      //   return `日志内容 ${this.data.content || ''}`
      // }
      return ''
    },

    /**
     * @description: 获取展示字段
     * formType relative 相关
     * hasDetails true 能查看  module 大模块 type 是小模块
     * @param {*}
     * @return {*}
     */
    getContentFields() {
      const { type, content, title } = this.data
      const contentObj = strToJSON(content) || {}
      if (type == 9) {
        // 日程
        return [
          { name: '日程内容', formType: 'text', value: contentObj.content },
          { name: '日程类型', formType: 'text', value: contentObj.type },
          { name: '起止时间', formType: 'text',
            value: `${timeToFormatTime(contentObj.startTime, 'YYYY-MM-DD HH:mm:ss')}-${timeToFormatTime(contentObj.endTime, 'YYYY-MM-DD HH:mm:ss')}` },
          { name: '关联业务', formType: 'relative', value: this.getRelativeList(contentObj) }
        ]
      } else if (type == 1 || type == 2 || type == 3) {
        // 任务 分配 邀请 完成
        return [
          { name: `${this.isTimeIterationTask ? '迭代名称' : '任务标题'}`, formType: 'text', value: title },
          { name: '开始时间', formType: 'text', value: contentObj.startTime },
          { name: '结束时间', formType: 'text', value: contentObj.endTime }
          // { name: '关联业务', formType: 'relative', value: this.getRelativeList(contentObj) }
        ]
      } else if ([10, 11, 26, 1000, 1004, 1008].includes(Number(type))) {
        // 合同 拒绝 通过 待审核 1000 恢复 1004 拒绝审批
        const customer = isObject(contentObj.customer) ? contentObj.customer : null
        const products = isArray(contentObj.products) ? contentObj.products : []
        const data = [
          { name: '合同名称', formType: 'text', value: title },
          { name: '合同金额', formType: 'text', value: this.getMoneyUnit(contentObj.money) },
          { name: '关联客户', module: 'crm', hasDetails: true, type: 'customer', id: customer ? customer.id : '', value: customer ? customer.name : '' },
          { name: '关联产品', formType: 'text', value: products.map(item => item.name).join('、') }
        ]
        if (type == 10) {
          data.push({ name: '拒绝理由', formType: 'text', value: contentObj.remarks })
        }
        return data
      } else if ([217, 218, 219, 220, 221, 222].includes(Number(type))) {
        // 报价单 拒绝 通过 待审核 1000 恢复 1004 拒绝审批
        const customer = isObject(contentObj.customer) ? contentObj.customer : null
        const products = isArray(contentObj.products) ? contentObj.products : []
        const data = [
          { name: '报价单名称', formType: 'text', value: title },
          { name: '报价单金额', formType: 'text', value: this.getMoneyUnit(contentObj.money) },
          { name: '关联客户', module: 'crm', hasDetails: true, type: 'customer', id: customer ? customer.id : '', value: customer ? customer.name : '' },
          { name: '关联产品', formType: 'text', value: products.map(item => item.name).join('、') }
        ]
        if (type == 219) {
          data.push({ name: '拒绝理由', formType: 'text', value: contentObj.remarks })
        }
        return data
      } else if ([12, 13, 27, 1001, 1005, 1009].includes(Number(type))) {
        // 回款 拒绝 通过 待审核
        const customer = isObject(contentObj.customer) ? contentObj.customer : null
        // const contract = isObject(contentObj.contract) ? contentObj.contract : null
        const contract = contentObj.contract || []
        const data = [
          { name: '回款编号', formType: 'text', value: title },
          { name: '回款金额', formType: 'text', value: this.getMoneyUnit(contentObj.money) },
          { name: '关联客户', module: 'crm', hasDetails: true, type: 'customer', id: customer ? customer.id : '', value: customer ? customer.name : '' },
          { name: '关联合同', module: 'crm', hasDetails: true, type: 'contract', value: contract }
        ]
        if (type == 12) {
          data.push({ name: '拒绝理由', formType: 'text', value: contentObj.remarks })
        }
        return data
      } else if ([35, 36, 37, 1002, 1006, 1010].includes(Number(type))) {
        // 发票 拒绝 通过 待审核
        const customer = isObject(contentObj.customer) ? contentObj.customer : null
        const contract = contentObj.contract || []
        const data = [
          { name: '发票编号', formType: 'text', value: title },
          { name: '开票金额', formType: 'text', value: this.getMoneyUnit(contentObj.money) },
          { name: '关联客户', module: 'crm', hasDetails: true, type: 'customer', id: customer ? customer.id : '', value: customer ? customer.name : '' },
          { name: '关联合同', module: 'crm', hasDetails: true, type: 'contract', value: contract },
          { name: '开票日期', formType: 'text', value: contentObj.createTime }
        ]
        if (type == 35) {
          data.push({ name: '拒绝理由', formType: 'text', value: contentObj.remarks })
        }
        return data
      } else if (type == 6 || type == 7 || type == 25 || type == 158) {
        // OA 拒绝 通过 待审核
        const oaType = contentObj.type
        let data = []
        if (oaType == 6) { // 借款
          data = [
            { name: '审批类型', formType: 'text', value: contentObj.examineName },
            { name: '借款事由', formType: 'text', value: contentObj.content },
            { name: '借款金额', formType: 'text', value: contentObj.money }
          ]
        } else if (oaType == 2) { // 请假
          data = [
            { name: '审批类型', formType: 'text', value: contentObj.examineName },
            { name: '请假事由', formType: 'text', value: contentObj.content },
            { name: '请假起止时间', formType: 'text',
              value: `${timeToFormatTime(contentObj.startTime, 'YYYY-MM-DD HH:mm:ss')}-${timeToFormatTime(contentObj.endTime, 'YYYY-MM-DD HH:mm:ss')}` },
            { name: '请假时长', formType: 'text', value: contentObj.duration }
          ]
        } else if (oaType == 3) { // 出差
          data = [
            { name: '审批类型', formType: 'text', value: contentObj.examineName },
            { name: '出差事由', formType: 'text', value: contentObj.content },
            { name: '出差总天数', formType: 'text', value: contentObj.duration }
          ]
        } else if (oaType == 4) { // 加班
          data = [
            { name: '审批类型', formType: 'text', value: contentObj.examineName },
            { name: '加班原因', formType: 'text', value: contentObj.content },
            { name: '加班起止时间', formType: 'text',
              value: `${timeToFormatTime(contentObj.startTime, 'YYYY-MM-DD HH:mm:ss')}-${timeToFormatTime(contentObj.endTime, 'YYYY-MM-DD HH:mm:ss')}` },
            { name: '加班总天数', formType: 'text', value: contentObj.duration }
          ]
        } else if (oaType == 5) { // 差旅
          data = [
            { name: '审批类型', formType: 'text', value: contentObj.examineName },
            { name: '差旅事由', formType: 'text', value: contentObj.content },
            { name: '报销总金额', formType: 'text', value: contentObj.money }
          ]
        } else {
          // 1 普通审批 和 自定义审批
          data = [
            { name: '审批类型', formType: 'text', value: contentObj.examineName },
            { name: '审批内容', formType: 'text', value: contentObj.content }
          ]
        }

        if (type == 6) {
          data.push({ name: '拒绝理由', formType: 'text', value: contentObj.remarks })
        }
        return data
      } else if (type == 99) {
        // 安排面试
        return [
          { name: '面试时间', formType: 'text', value: contentObj.createTime },
          { name: '面试轮次', formType: 'text', value: contentObj.content }
        ]
      } else if ([133, 134, 135, 1003, 1007, 1011].includes(type)) {
        const temps = [
          { name: '阶段名称', formType: 'text', value: contentObj.name }
        ]

        if (type === 135) temps.push({ name: '拒绝理由', formType: 'text', value: contentObj.remarks })
        return temps
      } else if ([6014, 6015].includes(type)) {
        // 工单任务的邀请、完成
        const priorityValue = {
          1: '很低',
          2: '低',
          3: '常规',
          4: '高',
          5: '很高'
        }[this.taskActivityContent.priority]
        return [
          { name: '任务名称', formType: 'text', value: this.taskActivityContent.taskName },
          { name: '到期日期', formType: 'text', value: this.taskActivityContent.expireDate },
          { name: '优先级', formType: 'text', value: priorityValue }
        ]
      } else if ([6016, 6017].includes(type)) {
        // 工单活动的邀请、完成
        const content = this.taskActivityContent
        const duration = `${content.durationHour ? content.durationHour + '小时' : ''}${content.durationMinute ? content.durationMinute + '分钟' : ''}`
        const priorityValue = {
          1: '很低',
          2: '低',
          3: '常规',
          4: '高',
          5: '很高'
        }[this.taskActivityContent.priority]
        return [
          { name: '活动名称', formType: 'text', value: this.taskActivityContent.activityName },
          { name: '类别', formType: 'text', value: { 1: '会议', 2: '展示' }[contentObj.type] },
          { name: '开始时间', formType: 'text', value: this.taskActivityContent.startTime },
          { name: '持续时间', formType: 'text', value: duration },
          { name: '优先级', formType: 'text', value: priorityValue }
        ]
      }
      return []
    },

    /**
     * @description: 获取金额及单位
     * @param {*} money
     * @return {*}
     */
    getMoneyUnit(money) {
      return separator(this.$money('crm', money || 0)) + this.$unit('crm')
    },

    /**
     * @description: 获取关联信息数据
     * @param {*}
     * @return {*}
     */
    getRelativeList(data) {
      const typeList = [{
        fieldName: 'customers',
        type: 'customer',
        label: '客户'
      }, {
        fieldName: 'contactss',
        type: 'contacts',
        label: '联系人'
      }, {
        fieldName: 'businesss',
        type: 'business',
        label: '商机'
      }, {
        fieldName: 'contracts',
        type: 'contract',
        label: '合同'
      }]
      const relativeList = []
      for (let index = 0; index < typeList.length; index++) {
        const typeItem = typeList[index]
        const list = data[typeItem.fieldName]
        if (isArray(list) && list.length > 0) {
          list.forEach(item => {
            relativeList.push({
              ...typeItem,
              ...item
            })
          })
        }
      }

      return relativeList
    }
  }
}
</script>

<style lang="scss" scoped>
.message-cell {
  position: relative;
  padding: 16px 24px 8px;
  font-size: 14px;
  line-height: 1.5;

  &-title {
    color: $--color-text-regular;
  }

  &-des {
    margin-top: 4px;
    color: $--color-text-primary;
  }

  &__hd {
    flex-shrink: 0;
    width: 28px;
    height: 28px;
    margin-right: 16px;
    line-height: 28px;
    text-align: center;
    background-color: $--color-n90;
    border-radius: 14px;

    .wk {
      font-size: 13px;
      color: white;
    }
  }

  &__bd {
    flex: 1;
    overflow: hidden;
  }

  &__ft {
    position: relative;
    flex-shrink: 0;
    width: 100px;
    margin-left: 24px;
    color: $--color-text-regular;
    text-align: right;

    .delete-btn {
      padding: 0;
      visibility: hidden;
    }

    ::v-deep .check-btn {
      display: flex;
      padding: 3px;
      cursor: auto;
      background-color: transparent;
      border-radius: 8px;

      span {
        display: inline-block;
        width: 8px;
        height: 8px;
        background-color: $--color-n90;
        border-radius: 4px;
      }

      &.is-unread {
        cursor: pointer;

        span {
          background-color: $--color-primary;
        }

        &:hover {
          background-color: rgba($color:$--color-b50, $alpha: 0.9);
        }
      }
    }
  }

  &:hover {
    background-color: #f7f8fa;

    .delete-btn {
      visibility: visible;
    }
  }
}

.click-content {
  color: $--color-primary;
  cursor: pointer;

  &:hover {
    text-decoration: underline;
  }
}

.is-invalid {
  color: $--color-text-secondary;
  pointer-events: none;
  cursor: initial;
}

.brief {
  margin-top: 4px;
  color: $--color-text-secondary;

  &-cell {
    display: flex;
    color: $--color-text-secondary;

    & + & {
      margin-top: 4px;
    }
  }

  &-label {
    flex-shrink: 0;
  }

  &-value {
    flex: 1;
    padding-left: 8px;

    &.is-visit {
      color: $--color-primary;
      cursor: pointer;
    }
  }
}

.relative {
  &-item {
    span:nth-child(2) {
      color: $--color-primary;
      cursor: pointer;
    }

    & + & {
      margin-top: 4px;
    }
  }
}
</style>
