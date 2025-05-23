<template>
    <div class="container">
        <div class="header">
            <h1></h1>
            <router-link to="/departments/create" class="btn btn-primary">Thêm phòng ban mới</router-link>
        </div>

        <div class="table-wrapper">
            <div class="table-responsive">
                <table class="table">
                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Tên phòng ban</th>
                            <th>Mô tả</th>
                            <th>Số vị trí</th>
                            <th>Thao tác</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="department in paginatedDepartments" :key="department.id">
                            <td>{{ department.id }}</td>
                            <td>{{ department.name }}</td>
                            <td>{{ department.description }}</td>
                            <td>{{ department.positions ? department.positions.length : 0 }}</td>
                            <td>
                                <div class="action-buttons">
                                    <router-link :to="`/positions?department=${department.id}`"
                                        class="btn btn-sm btn-info">Xem</router-link>
                                    <router-link :to="`/departments/${department.id}/edit`"
                                        class="btn btn-sm btn-edit btn-primary">Sửa</router-link>
                                    <button @click="confirmDelete(department)"
                                        class="btn btn-sm btn-danger">Xóa</button>
                                </div>
                            </td>
                        </tr>
                        <tr v-if="departments.length === 0">
                            <td colspan="5" class="text-center">Không có phòng ban nào</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>

        <!-- Phân trang -->
        <div class="pagination" v-if="departments.length > 0">
            <button @click="prevPage" :disabled="currentPage === 1" class="btn btn-page">
                Trang trước
            </button>
            <span>Trang {{ currentPage }} / {{ totalPages }}</span>
            <button @click="nextPage" :disabled="currentPage === totalPages" class="btn btn-page">
                Trang sau
            </button>
        </div>

        <!-- Modal xác nhận xóa -->
        <div class="modal" v-if="showDeleteModal">
            <div class="modal-content">
                <div class="modal-header">
                    <h2>Xác nhận xóa</h2>
                    <span class="close" @click="showDeleteModal = false">&times;</span>
                </div>
                <div class="modal-body">
                    <p>Bạn có chắc chắn muốn xóa phòng ban?</p>
                    <p class="warning" v-if="departmentToDelete?.positions && departmentToDelete.positions.length > 0">
                        Cảnh báo: Phòng ban này có {{ departmentToDelete.positions.length }} vị trí. Xóa phòng ban sẽ
                        xóa tất cả vị trí liên quan.
                    </p>
                </div>
                <div class="modal-footer">
                    <button @click="showDeleteModal = false" class="btn btn-secondary">Hủy</button>
                    <button @click="deleteDepartment" class="btn btn-danger">Xóa</button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import axios from 'axios';

export default {
    data() {
        return {
            departments: [],
            message: '',
            showDeleteModal: false,
            departmentToDelete: null,
            currentPage: 1,
            itemsPerPage: 10
        };
    },
    computed: {
        totalPages() {
            return Math.ceil(this.departments.length / this.itemsPerPage);
        },
        paginatedDepartments() {
            const start = (this.currentPage - 1) * this.itemsPerPage;
            const end = start + this.itemsPerPage;
            return this.departments.slice(start, end);
        }
    },
    created() {
        this.loadDepartments();

        // Kiểm tra nếu có message query param
        if (this.$route.query.message) {
            this.message = this.$route.query.message;
        }
    },
    methods: {
        prevPage() {
            if (this.currentPage > 1) {
                this.currentPage--;
            }
        },
        nextPage() {
            if (this.currentPage < this.totalPages) {
                this.currentPage++;
            }
        },
        async loadDepartments() {
            try {
                const response = await axios.get('/api/departments');
                this.departments = response.data;
            } catch (error) {
                console.error('Lỗi khi tải danh sách phòng ban:', error);
                this.message = 'Đã xảy ra lỗi khi tải danh sách phòng ban';
            }
        },
        confirmDelete(department) {
            this.departmentToDelete = department;
            this.showDeleteModal = true;
        },
        async deleteDepartment() {
            if (!this.departmentToDelete) return;

            try {
                await axios.delete(`/api/departments/${this.departmentToDelete.id}`);
                this.message = `Đã xóa phòng ban "${this.departmentToDelete.name}" thành công`;
                this.loadDepartments();
                this.showDeleteModal = false;
                this.departmentToDelete = null;
            } catch (error) {
                console.error('Lỗi khi xóa phòng ban:', error);
                if (error.response && error.response.data && error.response.data.message) {
                    this.message = error.response.data.message;
                } else {
                    this.message = 'Đã xảy ra lỗi khi xóa phòng ban';
                }
                this.showDeleteModal = false;
            }
        }
    }
};
</script>

<style scoped>
.container {
    max-width: 83vw;
    padding: 20px;
}

.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
}

.table-wrapper {
    position: relative;
    margin-top: 20px;
    overflow-x: auto;
    white-space: nowrap;
}

.table {
    width: 100%;
    min-width: 800px;
    border-collapse: collapse;
    background: #fff;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    border: 1px solid #ddd;
}

.table th,
.table td {
    padding: 12px 15px;
    text-align: left;
    border-bottom: 1px solid #ddd;
    border-right: 1px solid #ddd;
}

.table th:last-child,
.table td:last-child {
    border-right: none;
}

.table th {
    background-color: #f8f9fa;
    color: black;
    font-weight: bold;
    border-bottom: 2px solid #dee2e6;
}

.table tr:hover {
    background-color: #f5f5f5;
}

.action-buttons {
    display: flex;
    gap: 5px;
}

.btn {
    padding: 8px 12px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    text-decoration: none;
    display: inline-block;
}

.btn-primary {
    background-color: #ffc107;
    color: #333;
}

.btn-danger {
    background-color: #dc3545;
    color: white;
}

.btn-secondary {
    background-color: #6c757d;
    color: white;
}

.btn-info {
    background-color: #17a2b8;
    color: white;
}

.btn-sm {
    padding: 8px 16px;
    font-size: 14px;
}

.btn-primary {
    background-color: #007bff;
    color: white;
}

.btn-danger {
    background-color: #dc3545;
    color: white;
}

.btn-danger:hover {
    background-color: #c82333;
}

.btn-edit {
    background-color: #ffc107;
    color: #333;
}

.alert {
    padding: 12px 15px;
    margin-bottom: 20px;
    border-radius: 4px;
}

.alert-info {
    background-color: #d1ecf1;
    color: #0c5460;
    border: 1px solid #bee5eb;
}

.warning {
    color: #856404;
    background-color: #fff3cd;
    padding: 10px;
    border-radius: 4px;
    margin-top: 10px;
}

/* Modal styles */
.modal {
    display: block;
    position: fixed;
    z-index: 1;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    overflow: auto;
    background-color: rgba(0, 0, 0, 0.4);
}

.modal-content {
    background-color: #fefefe;
    margin: 15% auto;
    padding: 20px;
    border: 1px solid #888;
    width: 50%;
    border-radius: 5px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid #ddd;
    padding-bottom: 10px;
    margin-bottom: 15px;
}

.modal-footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    margin-top: 20px;
    padding-top: 15px;
    border-top: 1px solid #ddd;
}

.close {
    color: #aaa;
    font-size: 28px;
    font-weight: bold;
    cursor: pointer;
}

.close:hover {
    color: black;
}

@media (max-width: 768px) {
    .header {
        flex-direction: column;
        align-items: flex-start;
    }

    .header-actions {
        margin-top: 10px;
        width: 100%;
    }

    .modal-content {
        width: 90%;
    }
}

.table-responsive {
    overflow-x: auto;
    margin-bottom: 1rem;
    -webkit-overflow-scrolling: touch;
}

.pagination {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top: 20px;
    gap: 10px;
}

.btn-page {
    background-color: #007bff;
    color: white;
    padding: 8px 16px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.btn-page:disabled {
    background-color: #6c757d;
    cursor: not-allowed;
}

.btn-page:hover:not(:disabled) {
    background-color: #0056b3;
}

.pagination span {
    font-size: 14px;
}
</style>