<template>
  <section>
    <NavBar page="profile-page" />
    <section class="profile-main">
      <div class="container">
        <div class="columns profile-page is-multiline column">
          <div class="column is-3-desktop">
            <SideBar />
          </div>
          <div
            v-if="email.data && show"
            class="email-column column is-one-third"
          >
            <div class="email-name">
              {{ email.data.name }}
            </div>
            <div class="document-date-time">
              {{ new Date(email.data.created).toLocaleDateString("en-US") }}
              -
              {{ new Date(email.data.created).toLocaleTimeString("en-US") }}
            </div>
            <div class="email-size">Size: {{ email.data.size }}</div>
            <div class="email-category">
              Category: {{ email.data.category }}
            </div>
            <div class="email-owner">
              Owner: {{ email.data.first_name + " " + email.data.last_name }}
            </div>
            <div class="email-tags">
              Tags:
              <b-tag
                v-for="(tag, index) in email.data.tags"
                :key="index"
                type="is-info"
                >{{ tag }}</b-tag
              >
            </div>
            <div v-if="versions" class="email-versions">
              Versions: {{ versions.length }}
            </div>
            <div class="email-original">
              Original:
              <a target="blank" :href="encodeURI(email.data.originalFileUrl)"
                >View</a
              >
            </div>
          </div>
          <div v-if="show" class="upload-column column is-one-third">
            <b-field label="Upload a version of this file">
              <Upload :document="email" :email="true" @reload="getVersions" />
            </b-field>
          </div>
          <section v-if="show === false">
            <div class="column">
              <h1>You don't have access to this email</h1>
            </div>
          </section>
          <div
            v-if="email && show"
            class="column is-one-third view-file-column"
          >
            <b-button
              class="view-file-button"
              type="is-primary"
              @click="downloadVersion(version.data._id)"
              >View file</b-button
            >
          </div>
          <div v-if="show" class="column is-one-third view-file-column">
            <b-button class="view-file-button" type="is-primary is-light"
              >Compare file</b-button
            >
          </div>
          <div v-if="show" class="column is-one-third view-file-column">
            <b-button class="view-file-button" focused @click="openModal"
              ><b-icon icon="signature-freehand"></b-icon>&nbsp;&nbsp;&nbsp;
              Sign document</b-button
            >
          </div>
        </div>
        <section class="columns" v-if="!collaboratorFlag && show && versions">
          <div class="column is-one-third-desktop is-half-tablet">
            <Collaborate :version="versions.data" :page="true" />
          </div>
        </section>
        <section v-if="show" class="versions-section timeline">
          <div
            class="version-column"
            :class="index % 2 === 0 ? 'left' : 'right'"
            v-for="(version, index) in paginatedItems"
            :key="index"
          >
            <Email
              :email="version"
              :emailVersion="true"
              :index="(index - versions.length) * -1"
              :length="versions.length"
            />
          </div>
        </section>
        <section class="pagination-list columns">
          <div class="column">
            <b-pagination
              v-if="versions.length > 10"
              :total="versions.length"
              v-model="current"
              :range-before="rangeBefore"
              :range-after="rangeAfter"
              :order="order"
              :size="size"
              :simple="isSimple"
              :rounded="isRounded"
              :per-page="perPage"
              :icon-prev="prevIcon"
              :icon-next="nextIcon"
              aria-next-label="Next page"
              aria-previous-label="Previous page"
              aria-page-label="Page"
              aria-current-label="Current page"
              :page-input="hasInput"
              :page-input-position="inputPosition"
              :debounce-page-input="inputDebounce"
            >
            </b-pagination>
          </div>
        </section>
      </div>
    </section>
  </section>
</template>

<script>
import NavBar from "@/components/NavBar.vue";
import SideBar from "@/components/start-screen/SideBar.vue";
import Email from "@/components/open-email/Email.vue";
import Upload from "@/components/open-document/Upload.vue";
import FilesTable from "@/components/open-email/FilesTable.vue";
import Collaborate from "~/components/create-document/Collaborate.vue";
export default {
  middleware: "auth",
  name: "emailProfile",
  components: {
    NavBar,
    SideBar,
    FilesTable,
    Email,
    Upload,
    Collaborate,
  },
  data() {
    return {
      isEmpty: false,
      isBordered: false,
      isStriped: false,
      isNarrowed: false,
      isHoverable: false,
      isFocusable: false,
      isLoading: false,
      selected: null,
      email: {},
      versions: [],
      collaboratorsInput: [],
      collaboratorFlag: true,
      show: null,
      current: 1,
      perPage: 10,
      isComponentModalActive: false,
    };
  },
  computed: {
    collaborators: {
      get() {
        if (this.email.data) return this.email.data.collaborators;
      },
      set(values) {
        this.collaboratorsInput = values;
      },
    },
    paginatedItems() {
      const pageNumber = this.current - 1;
      if (this.versions.length > 0) {
        return this.versions.slice(
          pageNumber * this.perPage,
          (pageNumber + 1) * this.perPage
        );
      }
      return null;
    },
  },
  mounted() {
    this.getCollaborators();
  },
  methods: {
    openModal() {
      this.isComponentModalActive = true;
    },
    addDocStuff(data) {
      if (data) {
        const doc = {
          collaborators: this.collaborators,
        };
        this.$store.commit("doc/add", doc);
      }
    },
    async getCollaborators() {
      await this.$axios
        .$post(
          `${this.$config.app.backend_URL}/api/collaborators/${this.$nuxt.$route.params.id}`,
          {
            user_id: this.$auth.user._id,
            doc_type: "Email",
            email: this.$auth.user.email,
          }
        )
        .then((result) => {
          if (result.status === 200 && result.flag) {
            this.collaboratorFlag = true;
            this.show = true;
            this.getEmail();
            this.getVersions();
          }
          if (result.status === 201 && result.flag) {
            this.collaboratorFlag = false;
            this.show = true;
            this.getEmail();
            this.getVersions();
          }
        })
        .catch(() => {
          this.show = false;
        });
    },
    async getEmail() {
      this.email = await this.$axios.$post(
        `${this.$config.app.backend_URL}/api/emails/email/${this.$nuxt.$route.params.id}`
      );
    },
    async getVersions() {
      this.versions = await this.$axios.$post(
        `${this.$config.app.backend_URL}/api/email_versions/versions/${this.$nuxt.$route.params.id}`,
        {
          user_id: this.$auth.user._id,
        }
      );
      this.versions.reverse();
    },
  },
};
</script>
<style lang="scss" scoped>
.profile-main {
  .columns {
    padding-right: 0;
    margin-right: 0;
    padding-left: 0;
    margin-left: 0;
  }
  .profile-page {
    margin-top: 13px;
    .version-column {
      margin-top: 10px;
    }
    .email-name {
      font-size: 1.5rem;
      font-weight: 600;
    }
    .email-size {
      font-size: 1rem;
    }
    .tag {
      margin: 0 5px;
    }
  }
  .email-column {
    div {
      // margin: 10px 0;
    }
  }
  .versions-section {
    // background-color: #f46524;
  }
}
</style>
<style lang="scss">
//* The actual timeline (the vertical ruler) */
.timeline {
  position: relative;
  max-width: 1200px;
  margin: 0 auto;
}

/* The actual timeline (the vertical ruler) */
.timeline::after {
  content: "";
  position: absolute;
  width: 2px;
  background-color: lightgrey;
  top: 0;
  bottom: 0;
  left: 50%;
  margin-left: -3px;
}

/* Container around content */
.version-column {
  padding: 10px 40px;
  position: relative;
  background-color: inherit;
  width: 50%;
}

/* The circles on the timeline */
.version-column::after {
  content: "";
  position: absolute;
  width: 25px;
  height: 25px;
  right: -11px;
  background-color: #1c71e8;
  top: 20px;
  border-radius: 50%;
  z-index: 1;
}

/* Place the version-column.version-column to the left */
.left {
  left: 0;
}

/* Place the version-column.version-column to the right */
.right {
  left: 50%;
}

/* Add arrows to the left version-column.version-column (pointing right) */
.left::before {
  content: " ";
  height: 0;
  position: absolute;
  top: 22px;
  width: 0;
  z-index: 1;
  right: 30px;
  border: medium solid grey;
  border-width: 10px 0 10px 10px;
  border-color: transparent transparent transparent grey;
}

/* Add arrows to the right version-column.version-column (pointing left) */
.right::before {
  content: " ";
  height: 0;
  position: absolute;
  top: 22px;
  width: 0;
  z-index: 1;
  left: 30px;
  border: medium solid grey;
  border-width: 10px 10px 10px 0;
  border-color: transparent grey transparent transparent;
}

/* Fix the circle for version-column.version-columns on the right side */
.right::after {
  left: -14px;
}

/* The actual content */
.content {
  padding: 20px 30px;
  background-color: white;
  position: relative;
  border-radius: 6px;
}

/* Media queries - Responsive timeline on screens less than 600px wide */
@media screen and (max-width: 600px) {
  /* Place the timelime to the left */
  .timeline::after {
    left: 31px;
  }

  /* Full-width version-column.version-columns */
  .version-column {
    width: 100%;
    padding-left: 70px;
    padding-right: 25px;
  }

  /* Make sure that all arrows are pointing leftwards */
  .version-column::before {
    left: 60px;
    border: medium solid white;
    border-width: 10px 10px 10px 0;
    border-color: transparent grey transparent transparent;
  }

  /* Make sure all circles are at the same spot */
  .left::after,
  .right::after {
    left: 21px;
  }

  /* Make all right version-column.version-columns behave like the left ones */
  .right {
    left: 0%;
  }

  .version-column::after {
    width: 15px;
    height: 15px;
    top: 25px;
  }
}
</style>
