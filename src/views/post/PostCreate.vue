<template>
  <v-container>
    <v-card>
      <v-card-title>
        <v-row align="center">
          <v-col cols="auto">
            <v-select v-model="formData.type" :items="typeOptions" label="Type" required outlined dense
              :disabled="userRole === 'USER'" style="width: 200px; margin-top: 18px;" />
          </v-col>
          <v-spacer />
          <v-col cols="auto">
            <v-row justify="end" style="margin-right: 8px">
              <v-btn class="custom-btn" style="margin-right: 8px" @click="isModalOpen = true">기록</v-btn>
              <v-btn class="custom-btn" style="margin-right: 8px" @click="validateAndSubmit">저장</v-btn>
              <v-btn class="custom-btn" @click="showCancelConfirmationModal">취소</v-btn>
            </v-row>
          </v-col>
        </v-row>
      </v-card-title>
      <v-card-text>
        <v-row>
          <v-col cols="12">
            <v-text-field v-model="formData.title" :rules="title_rule" label="Title" required outlined dense />
          </v-col>
          <v-col cols="12">
            <quill-editor ref="quillEditorRef" v-model:value="formData.contents" :options="editorOptions"
              @blur="onEditorBlur" @focus="onEditorFocus" @ready="onEditorReady" class="custom-quill-editor" />
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>
    <!-- 기록 모달 컴포넌트 -->
    <ModalRecord v-if="isModalOpen" @close="isModalOpen = false" @insert="insertIntoEditor" />

    <!-- 제목 초과 경고 모달 -->
    <v-dialog v-model="showTitleLengthModal" persistent max-width="400px">
      <v-card>
        <v-card-text>{{ modalMessage }}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn class="action-button" text @click="closeTitleLengthModal">확인</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- 저장 확인 모달 -->
    <v-dialog v-model="showConfirmSaveModal" persistent max-width="400px">
      <v-card>
        <v-card-text>저장하시겠습니까?</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn class="action-button" text @click="submitForm">확인</v-btn>
          <v-btn class="action-button" text @click="closeConfirmSaveModal">취소</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- 취소 확인 모달 -->
    <v-dialog v-model="showCancelModal" persistent max-width="400px">
      <v-card>
        <v-card-text>작성한 내용은 저장되지 않습니다. <br> 목록으로 돌아가시겠습니까?</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn class="action-button" text @click="goBackToList">확인</v-btn>
          <v-btn class="action-button" text @click="closeCancelModal">취소</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

  </v-container>
</template>

<script>
import { quillEditor } from "vue3-quill";
import "quill/dist/quill.snow.css";
import { reactive, ref } from "vue";
import { useRouter } from "vue-router";
import axios from "axios";
import ModalRecord from './ModalRecord.vue';
import BlotFormatter from "quill-blot-formatter";
import Quill from "quill";
import interact from "interactjs";

// Register BlotFormatter as a module in Quill
Quill.register("modules/blotFormatter", BlotFormatter);

// Custom Video Blot 정의
const BlockEmbed = Quill.import("blots/block/embed");
class VideoBlot extends BlockEmbed {
  static create(value) {
    let node = super.create();
    node.setAttribute("controls", "");
    node.setAttribute("src", value);
    node.style.width = "320px";
    node.style.height = "180px";
    node.style.display = "block";
    node.classList.add("resizable-video");
    return node;
  }
  static value(node) {
    return node.getAttribute("src");
  }
}
VideoBlot.blotName = "video";
VideoBlot.tagName = "video";
Quill.register(VideoBlot);

export default {
  components: { quillEditor, ModalRecord },
  data() {
    return {
      title_rule: [v => v.length <= 50 || '50자 이하로 작성해주세요.',],
      showTitleLengthModal: false, // 제목 유효성 검사 모달 제어 변수
      showConfirmSaveModal: false, // 저장 확인 모달 제어 변수
      showCancelModal: false, // 취소 확인 모달 제어 변수
      modalMessage: "", // 모달에 표시될 메시지
    };
  },
  methods: {
    validateAndSubmit() {
      // 제목 유효성 검사
      if (!this.formData.title) {
        // 제목이 비어 있는 경우
        this.modalMessage = "제목을 입력해주세요.";
        this.showTitleLengthModal = true;
      } else if (this.formData.title.length > 50) {
        // 제목이 50자를 초과하는 경우
        this.modalMessage = "제목은 50자 이하로 작성해야 합니다.";
        this.showTitleLengthModal = true;
      } else {
        this.showConfirmSaveModal = true; // 유효성 검사를 통과하면 저장 확인 모달을 띄움      }
      }
    },
    closeTitleLengthModal() {
      this.showTitleLengthModal = false; // 모달 닫기
    },
    closeConfirmSaveModal() {
      this.showConfirmSaveModal = false; // 저장 확인 모달 닫기
    },
    showCancelConfirmationModal() {
      this.showCancelModal = true; // 취소 확인 모달 열기
    },
    closeCancelModal() {
      this.showCancelModal = false; // 취소 확인 모달 닫기
    },
    goBackToList() {
      this.showCancelModal = false; // 모달 닫기
      this.$router.push("/post/list"); // 목록 페이지로 이동
    },
  },
  setup() {
    const router = useRouter();
    const quillEditorRef = ref(null);
    const isModalOpen = ref(false);
    const quillInstance = ref(null);

    const formData = reactive({
      type: 'POST',
      title: "",
      contents: "",
    });

    const typeOptions = ["NOTICE", "POST"];
    const userRole = localStorage.getItem("role");
    const isLogin = !!localStorage.getItem("token");

    if (userRole === "USER") {
      formData.type = "POST";
    }

    const handleMediaUpload = async (acceptType) => {
      if (!quillInstance.value) {
        console.error("Quill Editor is not initialized yet.");
        return;
      }
      const fileInput = document.createElement("input");
      fileInput.setAttribute("type", "file");
      fileInput.setAttribute("accept", acceptType);
      fileInput.click();

      fileInput.onchange = async () => {
        const file = fileInput.files[0];
        const mediaUrl = await uploadMedia(file);

        if (mediaUrl) {
          const range = quillInstance.value.getSelection();
          if (range) {
            const extension = mediaUrl.split(".").pop().toLowerCase();
            const videoExtensions = ["mp4", "webm", "ogg"];

            if (videoExtensions.includes(extension)) {
              quillInstance.value.insertEmbed(range.index, "video", mediaUrl);
              makeResizable();
            } else {
              quillInstance.value.insertEmbed(range.index, "image", mediaUrl);
            }
          }
        }
      };
    };

    const uploadMedia = async (file) => {
      const formData = new FormData();
      formData.append("file", file);
      try {
        const response = await axios.post(
          "http://localhost:8090/post/upload-media",
          formData,
          { headers: { "Content-Type": "multipart/form-data" } }
        );
        return response.data.result;
      } catch (error) {
        console.error("Error uploading media:", error);
        return null;
      }
    };

    const makeResizable = () => {
      interact(".resizable-video").resizable({
        edges: { left: true, right: true, bottom: true, top: true },
        listeners: {
          move(event) {
            let { x, y } = event.target.dataset;
            x = (parseFloat(x) || 0) + event.deltaRect.left;
            y = (parseFloat(y) || 0) + event.deltaRect.top;
            Object.assign(event.target.style, {
              width: `${event.rect.width}px`,
              height: `${event.rect.height}px`,
              transform: `translate(${x}px, ${y}px)`,
            });
            event.target.dataset.x = x;
            event.target.dataset.y = y;
          },
        },
      });
    };

    const onEditorReady = (quill) => {
      quillInstance.value = quill; // Quill 인스턴스 저장
      console.log("Editor ready!", quill);
      makeResizable();
    };

    const editorOptions = {
      theme: "snow",
      placeholder: "Write your contents here...",
      modules: {
        toolbar: {
          container: [
            [{ header: [1, 2, false] }],
            ["bold", "italic", "underline"],
            [{ list: "ordered" }, { list: "bullet" }],
            ["link", "image", "video"],
          ],
          handlers: {
            image: () => handleMediaUpload("image/*"),
            video: () => handleMediaUpload("video/*"),
          },
        },
        blotFormatter: {},
      },
    };

    const onEditorBlur = (quill) => {
      console.log("Editor blur!", quill);
    };

    const onEditorFocus = (quill) => {
      console.log("Editor focus!", quill);
    };

    const submitForm = async () => {

      const data = new FormData();
      data.append("type", formData.type);
      data.append("title", formData.title);
      data.append("contents", formData.contents);
      try {
        const response = await axios.post("http://localhost:8090/post/create", data, {
          headers: { "Content-Type": "multipart/form-data" },
        });
        const createdPostId = response.data.result;
        router.push(`/post/detail/${createdPostId}`);
      } catch (error) {
        console.error("Error submitting form:", error);
      }
    };

    const cancel = () => {
      router.push("/post/list");
    };

    // 모달에서 전달된 데이터를 Quill Editor에 삽입
    const insertIntoEditor = (data) => {
      const quill = quillInstance.value; // Quill 인스턴스 가져오기

      // WOD 데이터 삽입
      if (data.wod) {
        let wodText = `
        WOD\n
        Time Cap: ${data.wod.timeCap}
        Rounds: ${data.wod.rounds}
        Info: ${data.wod.info}
        `;

        data.wod.wodDetResDtoList.forEach((detail) => {
          wodText += `
          ${detail.name}: ${detail.contents}
          `;
        });

        quill.insertText(quill.getLength(), wodText); // WOD 데이터를 먼저 삽입
      }

      // Record 데이터 삽입
      if (data.record) {
        const recordText = `
        Record\n
        SNF: ${data.record.snf === "Y" ? "성공" : "실패"}
        Time: ${data.record.time.slice(0, 5)}
        Comments: ${data.record.comments}
        `;

        quill.insertText(quill.getLength(), recordText); // Record 데이터를 나중에 삽입
      }

      isModalOpen.value = false; // 모달 닫기
    };

    return {
      formData,
      typeOptions,
      userRole,
      isLogin,
      quillEditorRef,
      editorOptions,
      onEditorBlur,
      onEditorFocus,
      onEditorReady,
      submitForm,
      cancel,
      isModalOpen,
      insertIntoEditor,
    };
  },
};
</script>


<style scoped>
.background-wrapper {
  background: linear-gradient(135deg, #1c1c1c, #3a3a3a);
  /* 그라데이션 배경 */
  padding: 20px;
  min-height: 100vh;
  /* 화면 전체 높이 */
}

.custom-quill-editor {
  min-height: 300px;
  /* 기본 높이 설정 */
}

.custom-btn {
  background-color: black !important;
  color: white !important;
  font-weight: 600;
  /* 글씨 두께를 조금 두껍게 설정 */
}

.v-card-title {
  padding: 0 16px;
}
</style>
