<template>

  <CoreBase
    :immersivePage="false"
    :authorized="userIsAuthorized"
    authorizedRole="adminOrCoach"
    :showSubNav="true"
  >
    <TopNavbar slot="sub-nav" />

    <KPageContainer>
      <PlanHeader />

      <div class="filter-and-button">
        <KSelect
          v-model="statusSelected"
          :label="$tr('show')"
          :options="statusOptions"
          :inline="true"
        />
        <KRouterLink
          :primary="true"
          appearance="raised-button"
          :to="newExamRoute"
          :text="$tr('newExam')"
        />
      </div>
      <CoreTable>
        <thead slot="thead">
          <tr>
            <th>{{ $tr('title') }}</th>
            <th>{{ $tr('assignedGroupsHeader') }}</th>
            <th>
              {{ $tr('status') }}
              <CoreInfoIcon
                :iconAriaLabel="$tr('statusDescription')"
                :tooltipText="$tr('statusTooltipText')"
                tooltipPlacement="bottom"
              />
            </th>
            <th><span class="visuallyhidden">{{ examReportPageStrings.$tr('options') }}</span></th>
          </tr>
        </thead>
        <transition-group slot="tbody" tag="tbody" name="list">
          <tr
            v-for="exam in filteredExams"
            :key="exam.id"
          >
            <td>
              <KLabeledIcon>
                <KIcon slot="icon" quiz />
                {{ exam.title }}
              </KLabeledIcon>
            </td>

            <td> {{ genRecipientsString(exam.groups) }} </td>

            <td>
              <QuizActive :active="exam.active" />
            </td>

            <td class="core-table-button-col">
              <KDropdownMenu
                slot="optionsDropdown"
                :text="examReportPageStrings.$tr('options')"
                :options="actionOptions"
                appearance="flat-button"
                @select="showModal($event, exam)"
              />
            </td>
          </tr>
        </transition-group>
      </CoreTable>

      <p v-if="!exams.length">
        {{ $tr('noExams') }}
      </p>
      <p v-else-if="statusSelected.value === $tr('activeExams') && !activeExams.length">
        {{ $tr('noActiveExams') }}
      </p>
      <p v-else-if=" statusSelected.value === $tr('inactiveExams') && !inactiveExams.length">
        {{ $tr('noInactiveExams') }}
      </p>
    </KPageContainer>

    <AssignmentDetailsModal
      v-if="showEditModal"
      ref="detailsModal"
      :modalTitle="manageExamModalStrings.$tr('editExamDetails')"
      :modalTitleErrorMessage="manageExamModalStrings.$tr('duplicateTitle')"
      :submitErrorMessage="manageExamModalStrings.$tr('saveExamError')"
      :showDescriptionField="false"
      :isInEditMode="true"
      :initialTitle="editExam.title"
      :initialSelectedCollectionIds="editExam.groups"
      :classId="classId"
      :groups="groups"
      :showActiveOption="true"
      :initialActive="editExam.active"
      :modalActiveText="manageExamModalStrings.$tr('changeExamStatusActive')"
      :modalInactiveText="manageExamModalStrings.$tr('changeExamStatusInactive')"
      @save="handleExamDetails"
      @cancel="showEditModal = false"
    />

    <AssignmentCopyModal
      v-if="showCopyModal"
      :modalTitle="manageExamModalStrings.$tr('copyExamTitle')"
      :assignmentQuestion="manageExamModalStrings.$tr('assignmentQuestion')"
      :classId="classId"
      :classList="classList"
      @copy="handleExamCopy"
      @cancel="showCopyModal = false"
    />

    <AssignmentDeleteModal
      v-if="showDeleteModal"
      :modalTitle="manageExamModalStrings.$tr('deleteExamTitle')"
      :modalDescription="manageExamModalStrings.$tr(
        'deleteExamDescription',
        { title: editExam.title }
      )"
      :modalConfirmation="manageExamModalStrings.$tr('deleteExamConfirmation')"
      @delete="handleExamDelete"
      @cancel="showDeleteModal = false"
    />

  </CoreBase>

</template>


<script>

  import find from 'lodash/find';
  import { mapState, mapActions, mapMutations } from 'vuex';
  import CoreTable from 'kolibri.coreVue.components.CoreTable';
  import KRouterLink from 'kolibri.coreVue.components.KRouterLink';
  import KSelect from 'kolibri.coreVue.components.KSelect';
  import CoreInfoIcon from 'kolibri.coreVue.components.CoreInfoIcon';
  import { ContentNodeKinds, ERROR_CONSTANTS } from 'kolibri.coreVue.vuex.constants';
  import { crossComponentTranslator } from 'kolibri.utils.i18n';
  import CatchErrors from 'kolibri.utils.CatchErrors';
  import KLabeledIcon from 'kolibri.coreVue.components.KLabeledIcon';
  import KIcon from 'kolibri.coreVue.components.KIcon';
  import { PageNames } from '../../../constants';
  import commonCoach from '../../common';
  import PlanHeader from '../../plan/PlanHeader';
  import AssignmentDetailsModal from '../../plan/assignments/AssignmentDetailsModal';
  import AssignmentCopyModal from '../../plan/assignments/AssignmentCopyModal';
  import AssignmentDeleteModal from '../../plan/assignments/AssignmentDeleteModal';
  import { AssignmentActions } from '../../../constants/assignmentsConstants';
  import QuizActive from '../../common/QuizActive';
  import ExamReportPage from './ExamReportPage';
  import ManageExamModals from './ManageExamModals';

  const examReportPageStrings = crossComponentTranslator(ExamReportPage);
  const manageExamModalStrings = crossComponentTranslator(ManageExamModals);

  export default {
    name: 'CoachExamsPage',
    $trs: {
      exams: 'Quizzes',
      allExams: 'All quizzes',
      activeExams: 'Active quizzes',
      inactiveExams: 'Inactive quizzes',
      newExam: 'New quiz',
      title: 'Title',
      assignedGroupsHeader: 'Visible to',
      noExams: 'You do not have any quizzes',
      noActiveExams: 'No active quizzes',
      noInactiveExams: 'No inactive quizzes',
      show: 'Show',
      status: 'Status',
      statusDescription: 'Status description',
      statusTooltipText: 'Learners can only see active quizzes',
      entireClass: 'Entire class',
      groups: '{count, number, integer} {count, plural, one {Group} other {Groups}}',
      nobody: 'Nobody',
      documentTitle: 'Quizzes',
    },
    metaInfo() {
      return {
        title: this.$tr('documentTitle'),
      };
    },
    components: {
      PlanHeader,
      CoreTable,
      KRouterLink,
      KSelect,
      CoreInfoIcon,
      AssignmentDetailsModal,
      AssignmentCopyModal,
      AssignmentDeleteModal,
      QuizActive,
      KLabeledIcon,
      KIcon,
    },
    mixins: [commonCoach],
    data() {
      return {
        statusSelected: { label: this.$tr('allExams'), value: this.$tr('allExams') },
        showEditModal: false,
        showCopyModal: false,
        showDeleteModal: false,
        editExam: {},
      };
    },
    computed: {
      ...mapState(['classList']),
      ...mapState('examsRoot', { fullExamInfo: 'exams' }),
      examReportPageStrings() {
        return examReportPageStrings;
      },
      manageExamModalStrings() {
        return manageExamModalStrings;
      },
      examKind() {
        return ContentNodeKinds.EXAM;
      },
      sortedExams() {
        return this.exams.slice().reverse();
      },
      statusOptions() {
        return [
          { label: this.$tr('allExams'), value: this.$tr('allExams') },
          { label: this.$tr('activeExams'), value: this.$tr('activeExams') },
          { label: this.$tr('inactiveExams'), value: this.$tr('inactiveExams') },
        ];
      },
      actionOptions() {
        return [
          {
            label: this.examReportPageStrings.$tr('editDetails'),
            value: AssignmentActions.EDIT_DETAILS,
          },
          {
            label: this.examReportPageStrings.$tr('copyExamOptionLabel'),
            value: AssignmentActions.COPY,
          },
          { label: this.examReportPageStrings.$tr('delete'), value: AssignmentActions.DELETE },
        ];
      },
      activeExams() {
        return this.sortedExams.filter(exam => exam.active === true);
      },
      inactiveExams() {
        return this.sortedExams.filter(exam => exam.active === false);
      },
      filteredExams() {
        const filter = this.statusSelected.label;
        if (filter === this.$tr('activeExams')) {
          return this.activeExams;
        } else if (filter === this.$tr('inactiveExams')) {
          return this.inactiveExams;
        }
        return this.sortedExams;
      },
      newExamRoute() {
        return { name: PageNames.EXAM_CREATION_ROOT };
      },
    },
    methods: {
      ...mapActions('examReport', ['updateExamDetails', 'copyExam', 'deleteExam']),
      ...mapMutations('classSummary', ['UPDATE_ITEM', 'CREATE_ITEM', 'DELETE_ITEM']),
      showModal(event, exam) {
        this.editExam = exam;
        if (event.value === AssignmentActions.EDIT_DETAILS) {
          this.showEditModal = true;
        } else if (event.value === AssignmentActions.COPY) {
          this.showCopyModal = true;
        }
        if (event.value === AssignmentActions.DELETE) {
          this.showDeleteModal = true;
        }
      },
      // format desired by the server, including class
      serverAssignmentPayload(listOfIDs) {
        const assignedToClass = listOfIDs.length === 0 || listOfIDs[0] === this.classId;
        if (assignedToClass) {
          return [{ collection: this.classId }];
        }
        return listOfIDs.map(id => {
          return { collection: id };
        });
      },
      // format used client-side, with only groups
      clientAssigmentState(listOfIDs) {
        const assignedToClass = listOfIDs.length === 0 || listOfIDs[0] === this.classId;
        if (assignedToClass) {
          return [];
        }
        return listOfIDs;
      },
      handleExamDetails(result) {
        const listOfIDs = result.assignments.map(item => item.collection);
        const serverPayload = {
          title: result.title,
          active: result.active,
          assignments: this.serverAssignmentPayload(listOfIDs),
        };
        this.updateExamDetails({ examId: this.editExam.id, payload: serverPayload })
          .then(() => {
            const object = {
              id: this.editExam.id,
              title: result.title,
              groups: this.clientAssigmentState(listOfIDs),
              active: result.active,
            };
            this.UPDATE_ITEM({ map: 'examMap', id: object.id, object });
            this.showEditModal = false;
          })
          .catch(error => {
            const errors = CatchErrors(error, [ERROR_CONSTANTS.UNIQUE]);
            if (errors) {
              this.$refs.detailsModal.handleSubmitTitleFailure();
            } else {
              this.$refs.detailsModal.handleSubmitFailure();
            }
          });
      },
      handleExamCopy(selectedClassroomId, listOfIDs) {
        const title = manageExamModalStrings
          .$tr('copyOfExam', { examTitle: this.editExam.title })
          .trim()
          .substring(0, 50)
          .trim();
        const currentExam = find(this.fullExamInfo, exam => exam.id === this.editExam.id);
        const exam = {
          collection: selectedClassroomId,
          title,
          question_count: currentExam.questionCount,
          question_sources: currentExam.questionSources,
          assignments: this.serverAssignmentPayload(listOfIDs),
        };
        const classroomName = find(this.classList, { id: selectedClassroomId }).name;

        this.copyExam({ exam, className: classroomName }).then(result => {
          // If exam was copied to the current classroom, add it to the classSummary module
          if (selectedClassroomId === this.classId) {
            const object = {
              id: result.id,
              title: result.title,
              groups: this.clientAssigmentState(listOfIDs),
              active: false,
            };
            this.CREATE_ITEM({ map: 'examMap', id: object.id, object });
          }
          this.showCopyModal = false;
        });
      },
      handleExamDelete() {
        this.deleteExam(this.editExam.id).then(() => {
          this.DELETE_ITEM({ map: 'examMap', id: this.editExam.id });
          this.showDeleteModal = false;
        });
      },
      genExamRoute(examId) {
        return {
          name: PageNames.EXAM_PREVIEW,
          params: { examId },
        };
      },
      genRecipientsString(groups) {
        if (!groups.length) {
          return this.$tr('entireClass');
        } else {
          return this.$tr('groups', { count: groups.length });
        }
      },
    },
  };

</script>


<style lang="scss" scoped>

  .filter-and-button {
    display: flex;
    flex-wrap: wrap-reverse;
    justify-content: space-between;
    button {
      align-self: flex-end;
    }
  }

  .center-text {
    text-align: center;
  }

</style>
