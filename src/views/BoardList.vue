<template>
  <section>
    <div class="page-title">
      <div>
        <h1>게시판</h1>
        <p>Spring Boot + MyBatis 게시판</p>
      </div>
      <router-link to="/boards/write" class="btn primary">글쓰기</router-link>
    </div>

    <div class="search-box">
      <select v-model="searchType">
        <option value="title">제목</option>
        <option value="content">내용</option>
        <option value="writer">작성자</option>
        <option value="titleContent">제목+내용</option>
      </select>
      <input
        v-model="keyword"
        @keyup.enter="search"
        placeholder="검색어를 입력하세요"
      />
      <button class="btn" @click="search">검색</button>
    </div>

    <div v-if="loading" class="state">게시글을 불러오는 중입니다...</div>
    <div v-else-if="error" class="state error">{{ error }}</div>

    <div v-else class="table-wrap">
      <table>
        <thead>
          <tr>
            <th class="num">번호</th>
            <th>제목</th>
            <th class="file-col">첨부</th>
            <th class="writer">작성자</th>
            <th class="date">작성일</th>
            <th class="views">조회</th>
          </tr>
        </thead>
        <tbody>
          <tr v-if="boards.length === 0">
            <td colspan="6" class="empty">게시글이 없습니다.</td>
          </tr>
          <tr v-for="board in boards" :key="board.boardId">
            <td>{{ board.boardId }}</td>
            <td class="subject">
              <router-link :to="`/boards/${board.boardId}`">
                {{ board.title }}
              </router-link>
            </td>
            <!-- 첨부파일 표시 (컬럼 하나로 통합) -->
            <td class="file-col">
              <span
                v-if="getFileCount(board) > 0"
                class="file-badge"
                title="첨부파일 있음"
              >
                📎 <small>{{ getFileCount(board) }}</small>
              </span>
              <span v-else class="no-file">-</span>
            </td>
            <td>{{ board.writer }}</td>
            <td>{{ formatDate(board.createdAt) }}</td>
            <td>{{ board.readCnt ?? 0 }}</td>
          </tr>
        </tbody>
      </table>
    </div>

    <Pagination :page="page" :total-pages="totalPages" @change="changePage" />
  </section>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { getBoards } from "../api/boardApi";
import Pagination from "../components/Pagination.vue";

const boards = ref([]);
const page = ref(1);
const recordSize = 10; // 한 페이지당 게시글 수
const totalPages = ref(1);
const loading = ref(false);
const error = ref("");
const searchType = ref("title");
const keyword = ref("");

// 첨부파일 개수 감지 헬퍼 (문자열로 와도 안전하게 숫자 변환)
function getFileCount(board) {
  if (!board) return 0;

  if (board.fileCount !== undefined && board.fileCount !== null) {
    const n = Number(board.fileCount);
    return Number.isNaN(n) ? 0 : n;
  }
  if (Array.isArray(board.fileList)) return board.fileList.length;
  if (Array.isArray(board.files)) return board.files.length;
  return 0;
}

async function loadBoards() {
  loading.value = true;
  error.value = "";

  try {
    // ✅ 백엔드 SearchDto/매퍼는 recordSize를 사용하므로 이름을 맞춰서 전송
    const { data } = await getBoards({
      page: page.value,
      recordSize,
      searchType: searchType.value,
      keyword: keyword.value,
    });

    boards.value = data.list ?? data.content ?? data.boards ?? [];

    // ✅ 백엔드는 totalPages가 아니라 totalCount/recordSize를 반환하므로 직접 계산
    const totalCount = data.totalCount ?? 0;
    const size = data.recordSize ?? recordSize;
    totalPages.value = Math.max(1, Math.ceil(totalCount / size));
  } catch (e) {
    error.value = e.response?.data?.message || "게시글을 불러오지 못했습니다.";
  } finally {
    loading.value = false;
  }
}

function search() {
  page.value = 1;
  loadBoards();
}

function changePage(p) {
  page.value = p;
  loadBoards();
}

function formatDate(value) {
  if (!value) return "";
  return String(value).substring(0, 10);
}

onMounted(loadBoards);
</script>

<style scoped>
.file-col {
  width: 60px;
  text-align: center;
}

.file-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 2px;
  color: #2b6cb0;
  font-weight: bold;
}

.no-file {
  color: #ccc;
}
</style>
