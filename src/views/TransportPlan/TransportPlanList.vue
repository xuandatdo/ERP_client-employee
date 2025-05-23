<template>
    <div class="container">
        <div class="search-wrapper">
            <div class="search-container">
                <input v-model="filters.search" type="text" placeholder="Tìm kiếm theo mã lô hàng, tài xế, biển số..."
                    class="search-input" @input="handleSearch" />
            </div>
            <div class="filter-container">
                <select v-model="filters.status" @change="handleSearch" class="filter-select">
                    <option value="">Tất cả trạng thái</option>
                    <option value="preparing">Đang chuẩn bị</option>
                    <option value="in_transit">Đang vận chuyển</option>
                    <option value="completed">Hoàn thành</option>
                    <option value="delayed">Chậm trễ</option>
                </select>
                <input type="date" v-model="filters.date" @change="handleSearch" class="filter-date" />
            </div>
            <router-link to="/transport/create" class="btn btn-add">Thêm kế hoạch vận chuyển</router-link>
        </div>

        <div class="table-wrapper">
            <table class="transport-table" v-if="paginatedPlans.length > 0">
                <thead>
                    <tr>
                        <th>Mã LH</th>
                        <th>Biển số xe</th>
                        <th>Tên tài xế</th>
                        <th>SĐT tài xế</th>
                        <th>Thời gian dự kiến</th>
                        <th>Trạng thái</th>
                        <th>Điểm giao hàng</th>
                        <th>Điểm lấy hàng</th>
                        <th>Số lượng</th>
                        <th>Thao tác</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="plan in paginatedPlans" :key="plan.id">
                        <td>{{ "LH" + plan.id }}</td>
                        <td>{{ truncate(plan.license_plate, 15) }}</td>
                        <td>{{ truncate(plan.driver_name, 20) }}</td>
                        <td>{{ plan.driver_phone }}</td>
                        <td>{{ formatDateTime(plan.expected_time) }}</td>
                        <td>
                            <div class="status-wrapper">
                                <select v-model="plan.status" @change="updateStatus(plan.id, plan.status)"
                                    :class="['status-select', getStatusClass(plan.status)]">
                                    <option value="preparing">Đang chuẩn bị</option>
                                    <option value="in_transit">Đang vận chuyển</option>
                                    <option value="completed">Hoàn thành</option>
                                    <option value="delayed">Chậm trễ</option>
                                </select>
                            </div>
                        </td>
                        <td>{{ truncate(plan.delivery_location, 15) }}</td>
                        <td>{{ truncate(plan.pickup_location, 15) }}</td>
                        <td>{{ plan.quantity }}</td>
                        <td>
                            <router-link :to="'/transport/' + plan.id + '/edit'" class="btn btn-edit">Sửa</router-link>
                            <button @click="confirmDelete(plan)" class="btn btn-delete">Xóa</button>
                        </td>
                    </tr>
                </tbody>
            </table>
            <div v-else class="no-data">Không có kế hoạch vận chuyển nào</div>
        </div>

        <div class="pagination">
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
                    <span class="close" @click="showDeleteModal = false">×</span>
                </div>
                <div class="modal-body">
                    <p>Bạn có chắc chắn muốn xóa kế hoạch vận chuyển này?</p>
                </div>
                <div class="modal-footer">
                    <button @click="showDeleteModal = false" class="btn btn-secondary">Hủy</button>
                    <button @click="deletePlan" class="btn btn-danger">Xóa</button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import axios from 'axios';
import { useToast } from 'vue-toastification';

export default {
    setup() {
        const toast = useToast();
        return { toast };
    },
    data() {
        return {
            transportPlans: [],
            filters: {
                search: '',
                status: '',
                date: '',
            },
            currentPage: 1,
            itemsPerPage: 10,
            showDeleteModal: false,
            planToDelete: null,
        };
    },
    mounted() {
        this.loadTransportPlans();
    },
    computed: {
        filteredPlans() {
            if (!this.filters.search && !this.filters.status && !this.filters.date) {
                return this.transportPlans;
            }

            return this.transportPlans.filter((plan) => {
                const query = this.filters.search.toLowerCase();
                const planId = `lh${plan.id}`.toLowerCase();
                const licensePlate = plan.license_plate.toLowerCase();
                const driverName = plan.driver_name.toLowerCase();
                const driverPhone = plan.driver_phone.toLowerCase();
                const deliveryLocation = plan.delivery_location.toLowerCase();
                const pickupLocation = plan.pickup_location.toLowerCase();

                const matchesSearch =
                    !this.filters.search ||
                    planId.includes(query) ||
                    licensePlate.includes(query) ||
                    driverName.includes(query) ||
                    driverPhone.includes(query) ||
                    deliveryLocation.includes(query) ||
                    pickupLocation.includes(query);

                const matchesStatus = !this.filters.status || plan.status === this.filters.status;

                const matchesDate = !this.filters.date || this.isSameDate(plan.expected_time, this.filters.date);

                return matchesSearch && matchesStatus && matchesDate;
            });
        },
        totalPages() {
            return Math.ceil(this.filteredPlans.length / this.itemsPerPage);
        },
        paginatedPlans() {
            const start = (this.currentPage - 1) * this.itemsPerPage;
            const end = start + this.itemsPerPage;
            return this.filteredPlans.slice(start, end);
        },
    },
    watch: {
        'filters.search'() {
            this.currentPage = 1;
        },
        'filters.status'() {
            this.currentPage = 1;
        },
        'filters.date'() {
            this.currentPage = 1;
        },
        totalPages(newVal) {
            if (this.currentPage > newVal && newVal > 0) {
                this.currentPage = newVal;
            }
        }
    },
    methods: {
        async loadTransportPlans() {
            try {
                const response = await axios.get('/api/transport-plans');
                this.transportPlans = response.data.data;

                // Kiểm tra và điều chỉnh trang hiện tại nếu cần
                if (this.totalPages > 0 && this.currentPage > this.totalPages) {
                    this.currentPage = this.totalPages;
                }
            } catch (error) {
                console.error('Lỗi khi tải danh sách kế hoạch vận chuyển:', error);
                this.toast.error('Có lỗi xảy ra khi tải danh sách');
            }
        },
        handleSearch: debounce(function () {
            this.currentPage = 1;
            this.loadTransportPlans();
        }, 300),
        confirmDelete(plan) {
            this.planToDelete = plan;
            this.showDeleteModal = true;
        },
        async deletePlan() {
            if (!this.planToDelete) return;

            try {
                const response = await axios.delete(`/api/transport-plans/${this.planToDelete.id}`);

                // Kiểm tra nếu là item cuối cùng của trang và không phải trang 1
                const isLastItemOfPage = this.paginatedPlans.length === 1 && this.currentPage > 1;

                await this.loadTransportPlans();

                // Nếu là item cuối cùng và xóa thành công, chuyển về trang trước
                if (isLastItemOfPage) {
                    this.currentPage--;
                }

                this.toast.success('Xóa kế hoạch vận chuyển thành công!');
                this.showDeleteModal = false;
                this.planToDelete = null;
            } catch (error) {
                console.error('Lỗi khi xóa kế hoạch vận chuyển:', error);

                let errorMessage = 'Có lỗi xảy ra khi xóa kế hoạch vận chuyển';

                if (error.response) {
                    console.log('Error status:', error.response.status);
                    console.log('Error data:', error.response.data);

                    if (error.response.status === 403) {
                        errorMessage = 'Bạn không có quyền xóa kế hoạch vận chuyển này';
                    } else if (error.response.status === 409) {
                        errorMessage = 'Không thể xóa: Kế hoạch vận chuyển đang được sử dụng';
                    } else if (error.response.data && error.response.data.message) {
                        errorMessage = error.response.data.message;
                    }
                }

                this.toast.error(errorMessage);
                this.showDeleteModal = false;
            }
        },
        truncate(text, maxLength) {
            if (!text || text.length <= maxLength) return text;
            return text.substring(0, maxLength) + '...';
        },
        formatDateTime(datetime) {
            if (!datetime) return '';
            try {
                const date = new Date(datetime);
                if (isNaN(date.getTime())) return ''; // Invalid date

                const year = date.getFullYear();
                const month = String(date.getMonth() + 1).padStart(2, '0');
                const day = String(date.getDate()).padStart(2, '0');
                const hours = String(date.getHours()).padStart(2, '0');
                const minutes = String(date.getMinutes()).padStart(2, '0');

                return `${day}/${month}/${year} ${hours}:${minutes}`;
            } catch (error) {
                console.error('Error formatting date:', error);
                return '';
            }
        },
        getStatusText(status) {
            const statusMap = {
                preparing: 'Đang chuẩn bị',
                in_transit: 'Đang vận chuyển',
                completed: 'Hoàn thành',
                delayed: 'Chậm trễ',
            };
            return statusMap[status] || status;
        },
        getStatusClass(status) {
            const classMap = {
                preparing: 'status-preparing',
                in_transit: 'status-in-transit',
                completed: 'status-completed',
                delayed: 'status-delayed',
            };
            return classMap[status] || '';
        },
        prevPage() {
            if (this.currentPage > 1) {
                this.currentPage--;
                console.log('Chuyển đến trang:', this.currentPage);
            }
        },
        nextPage() {
            if (this.currentPage < this.totalPages) {
                this.currentPage++;
                console.log('Chuyển đến trang:', this.currentPage);
            }
        },
        async updateStatus(planId, newStatus) {
            try {
                const response = await axios.patch(`/api/transport-plans/${planId}`, {
                    _method: 'PATCH',
                    status: newStatus,
                    ...this.transportPlans.find(plan => plan.id === planId)
                });

                if (response.data.success) {
                    this.toast.success('Cập nhật trạng thái thành công!');
                    await this.loadTransportPlans();
                }
            } catch (error) {
                console.error('Lỗi khi cập nhật trạng thái:', error);

                if (error.response && error.response.status === 422) {
                    const validationErrors = error.response.data.errors;
                    let errorMessage = 'Lỗi validation: ';

                    if (validationErrors) {
                        errorMessage += Object.values(validationErrors).flat().join(', ');
                    }

                    this.toast.error(errorMessage);
                } else {
                    this.toast.error('Có lỗi xảy ra khi cập nhật trạng thái');
                }

                await this.loadTransportPlans();
            }
        },
        isSameDate(dateTime1, dateTime2) {
            if (!dateTime1 || !dateTime2) return false;

            const date1 = new Date(dateTime1);
            const date2 = new Date(dateTime2);

            return date1.getFullYear() === date2.getFullYear() &&
                date1.getMonth() === date2.getMonth() &&
                date1.getDate() === date2.getDate();
        },
    },
};

function debounce(fn, delay) {
    let timeoutId;
    return function (...args) {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => fn.apply(this, args), delay);
    };
}
</script>

<style scoped>
.container {
    max-width: 83vw;
    padding: 20px;
}

.search-wrapper {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 20px;
}

.search-container {
    position: relative;
    width: 350px;
    transition: all 0.3s ease;
}

.search-input {
    width: 100%;
    padding: 12px 40px 12px 15px;
    border: 2px solid #ddd;
    border-radius: 25px;
    font-size: 14px;
    background-color: #fff;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
    transition: all 0.3s ease;
}

.search-input:hover {
    background-color: #f8f9fa;
}

.search-input:focus {
    outline: none;
    border-color: #007bff;
    box-shadow: 0 0 8px rgba(0, 123, 255, 0.3);
    width: 105%;
}

.filter-container {
    display: flex;
    gap: 10px;
    align-items: center;
}

.filter-select,
.filter-date {
    padding: 10px;
    border: 2px solid #ddd;
    border-radius: 4px;
    font-size: 14px;
    background-color: #fff;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
}

.filter-select:focus,
.filter-date:focus {
    outline: none;
    border-color: #007bff;
}

.btn {
    padding: 8px 16px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    text-decoration: none;
    display: inline-block;
    font-size: 14px;
}

.btn-add {
    background-color: #007bff;
    color: white;
    padding: 10px 20px;
    border-radius: 4px;
    transition: all 0.3s ease;
}

.btn-edit {
    background-color: #ffc107;
    color: #333;
    margin-right: 10px;
}

.btn-edit:hover {
    background-color: #e0a800;
}

.btn-delete {
    background-color: #dc3545;
    color: white;
}

.btn-delete:hover {
    background-color: #c82333;
}

.table-wrapper {
    position: relative;
    margin-top: 20px;
    overflow-x: auto;
    white-space: nowrap;
}

.transport-table {
    width: 100%;
    min-width: 1300px;
    border-collapse: collapse;
    background: #fff;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    border: 1px solid #ddd;
}

.transport-table th,
.transport-table td {
    padding: 12px 15px;
    text-align: left;
    border-bottom: 1px solid #ddd;
    border-right: 1px solid #ddd;
}

.transport-table th:last-child,
.transport-table td:last-child {
    border-right: none;
}

.transport-table th {
    background-color: #f8f9fa;
    color: black;
    font-weight: bold;
    border-bottom: 2px solid #dee2e6;
}

.transport-table tr:hover {
    background-color: #f5f5f5;
}

.status-badge {
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 12px;
    font-weight: bold;
}

.status-preparing {
    background-color: #ffc107;
    color: #000;
}

.status-in-transit {
    background-color: #17a2b8;
    color: white;
}

.status-completed {
    background-color: #28a745;
    color: white;
}

.status-delayed {
    background-color: #dc3545;
    color: white;
}

.pagination {
    margin-top: 20px;
    text-align: center;
}

.btn-page {
    background-color: #007bff;
    color: white;
    margin: 0 10px;
}

.btn-page:disabled {
    background-color: #6c757d;
    cursor: not-allowed;
}

.btn-page:hover:not(:disabled) {
    background-color: #0056b3;
}

span {
    font-size: 16px;
    margin: 0 10px;
}

.no-data {
    text-align: center;
    padding: 20px;
    color: #6c757d;
    font-size: 16px;
}

/* Modal styles */
.modal {
    display: block;
    position: fixed;
    z-index: 1000;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
}

.modal-content {
    background-color: #fff;
    margin: 15% auto;
    padding: 20px;
    border-radius: 5px;
    width: 400px;
    position: relative;
    animation: slideIn 0.3s ease-out;
}

@keyframes slideIn {
    from {
        transform: translateY(-100px);
        opacity: 0;
    }

    to {
        transform: translateY(0);
        opacity: 1;
    }
}

.modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid #ddd;
    padding-bottom: 10px;
    margin-bottom: 20px;
}

.modal-header h2 {
    margin: 0;
    font-size: 1.5rem;
    color: #333;
}

.close {
    font-size: 28px;
    font-weight: bold;
    color: #666;
    cursor: pointer;
}

.close:hover {
    color: #000;
}

.modal-body {
    margin-bottom: 20px;
}

.modal-body p {
    margin: 0;
    font-size: 1rem;
    color: #555;
}

.modal-footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    border-top: 1px solid #ddd;
    padding-top: 15px;
}

.modal-footer button {
    padding: 8px 20px;
    border-radius: 4px;
    font-size: 0.9rem;
    cursor: pointer;
    transition: all 0.3s ease;
}

.btn-secondary {
    background-color: #6c757d;
    color: white;
    border: none;
}

.btn-secondary:hover {
    background-color: #5a6268;
}

.btn-danger {
    background-color: #dc3545;
    color: white;
    border: none;
}

.btn-danger:hover {
    background-color: #c82333;
}

.status-wrapper {
    position: relative;
}

.status-select {
    padding: 6px 12px;
    border-radius: 20px;
    border: none;
    font-size: 14px;
    cursor: pointer;
    appearance: none;
    width: 100%;
    text-align: center;
    color: white;
}

.status-select.preparing {
    background-color: #ffc107;
}

.status-select.in_transit {
    background-color: #17a2b8;
}

.status-select.completed {
    background-color: #28a745;
}

.status-select.delayed {
    background-color: #dc3545;
}

.status-select:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(0, 123, 255, 0.25);
}

.status-wrapper::after {
    content: '▼';
    font-size: 12px;
    color: white;
    position: absolute;
    right: 10px;
    top: 50%;
    transform: translateY(-50%);
    pointer-events: none;
}
</style>