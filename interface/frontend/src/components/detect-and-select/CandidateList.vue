<template>
  <div class="container">
    <template>
      <div class="row">
        <div class="col-md-3">
          <div id="accordion">
            <template v-for="(candidate, index) in candidates">
              <div class="card">
                <div class="card-header cursor-pointer" @click="toggleShow(index)">
                  <p class="mb-0" :class="{ 'candidate-dismissed': candidate.review_result === REVIEW_RESULT.DISMISSED,
                      'candidate-marked': candidate.review_result === REVIEW_RESULT.MARKED}">
                    <span class="candidate-title">
                      Candidate {{ index + 1 }}
                    </span> (p={{ candidate.probability_concerning.toFixed(3) }})
                    <i class="pull-right" v-if="candidate._saving">Saving...</i>
                  </p>
                </div>

                <div class="collapse" :class="{ show: selectedCandidateIndex == index }">
                  <div class="card-block">
                    <candidate :candidate="candidate" :index="index"></candidate>
                    <a @click="markOrDismiss(candidate, REVIEW_RESULT.DISMISSED)">
                      <button type="button" class="btn btn-sm btn-secondary">Dismiss</button>
                    </a>
                    <a @click="markOrDismiss(candidate, REVIEW_RESULT.MARKED)">
                      <button type="button" class="btn btn-sm btn-danger">Mark concerning</button>
                    </a>
                  </div>
                </div>
              </div>

            </template>
          </div>
        </div>
        <div class="col-md-9" style="min-height: 500px">
          <open-dicom :view="viewerData" :marker="marker"></open-dicom>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
  import Vue from 'vue'
  import {EventBus} from '../../main.js'
  import Candidate from './Candidate'
  import OpenDicom from '../common/OpenDICOM'

  export default {
    components: {Candidate, OpenDicom},
    data () {
      return {
        REVIEW_RESULT: this.$constants.CANDIDATE_REVIEW_RESULT,
        selectedCandidateIndex: 0,
        lastViewedSeriesId: null
      }
    },
    computed: {
      candidates () {
        return this.$store.getters.candidates.sort((a, b) => {
          return b.probability_concerning - a.probability_concerning
        })
      },
      viewerData () {
        return {
          type: 'DICOM',
          prefixCS: ':/',
          prefixUrl: '/api/images/preview?dicom_location=',
          paths: this.$store.getters.imagePaths,
          sliceIndex: this.selectedCandidateIndex || 0
        }
      },
      marker () {
        if (!this.selectedCandidate) {
          return null
        }

        // extract centroid location
        const {x, y, z} = this.selectedCandidate.centroid

        return {x, y, z}
      },
      selectedCandidate () {
        // selected candidate is changing
        const candidate = this.candidates[this.selectedCandidateIndex] || null

        if (candidate) {
          this.viewerData.sliceIndex = candidate.centroid.z

          // to avoid screen refreshes - reinit viewer only when new candidate comes from other case
          if (this.lastViewedCaseUrl !== candidate.case.url) {
            this.viewerData.paths = candidate._filesPaths
          }

          this.lastViewedCaseUrl = candidate.case.url
        } else {
          this.lastViewedCaseUrl = null
        }

        return candidate
      }
    },
    mounted: function () {
      EventBus.$on('nodule-marker-moved', (x, y, z) => {
        if (!this.selectedCandidate) {
          console.error('can\'t save new coordinates - selectedCanidate is not found')
          return
        }

        const candidates = this.candidates
        const selectedCandidate = this.selectedCandidate
        const selectedCandidateIndex = this.selectedCandidateIndex

        // mark current candidate as saving
        selectedCandidate._saving = true

        // send data to backend
        this.$axios
            .post(selectedCandidate.url + 'move', {x, y, z})
            .then((response) => {
              if (response.status === 200) {
                Vue.set(candidates, selectedCandidateIndex, response.data)
              }
            })
      })
    },
    methods: {
      toggleShow (index) {
        this.selectedCandidateIndex = this.selectedCandidateIndex === index ? -1 : index
      },
      markOrDismiss (candidate, result) {
        candidate._saving = true

        this.$axios.patch(candidate.url, {review_result: result})
            .then((response) => {
              if (response.status === 200) {
                candidate._saving = false
                candidate.review_result = result
              } else {
                console.log('saving review result failed with response: ', response)
              }
            })
            .catch(() => {
              // TODO: error callback
            })
      }
    }
  }
</script>

<style lang="scss" scoped>
  .candidate-dismissed {
    opacity: 0.5;
  }

  .candidate-marked .candidate-title {
    font-weight: bold;
  }
</style>
