<!--hotelReservation\src\routes\+page.svelte-->
<script lang="ts">
    import { onMount } from 'svelte';

    // Types
    interface Reservation {
        id?: number;
        roomNumber: number;
        guestName: string;
        startDate: string;
        endDate: string;
        payment: number;
        totalAmount: number;
        status: 'occupied' | 'vacant';
    }

    interface Room {
        number: number;
        status: 'occupied' | 'vacant';
        reservation: Reservation | null;
    }

    // State
    let rooms: Room[] = Array.from({ length: 10 }, (_, i) => ({
        number: i + 1,
        status: 'vacant',
        reservation: null
    }));

    let formData = {
        guestName: '',
        startDate: '',
        startTime: '',
        endDate: '',
        endTime: '',
        payment: 0
    };

    // Calculate total amount based on days
    function calculateAmount(start: string, end: string): number {
        const startDate = new Date(start);
        const endDate = new Date(end);
        const diffTime = Math.abs(endDate.getTime() - startDate.getTime());
        const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
        return diffDays * 500; // 500 PHP per day
    }

    // Format date for display
    function formatDateTime(dateStr: string): string {
        return new Date(dateStr).toLocaleString('en-US', {
            year: 'numeric',
            month: 'short',
            day: 'numeric',
            hour: '2-digit',
            minute: '2-digit'
        });
    }

    // Update the fetch URLs to match your API endpoint
    const API_URL = 'http://localhost/php-hotel-reservation-api/reservations';

    // Handle form submission
    async function handleSubmit() {
        const startDateTime = `${formData.startDate}T${formData.startTime}`;
        const endDateTime = `${formData.endDate}T${formData.endTime}`;
        const totalAmount = calculateAmount(startDateTime, endDateTime);

        const reservation = {
            guestName: formData.guestName,
            startDate: startDateTime,
            endDate: endDateTime,
            payment: formData.payment,
            totalAmount
        };

        try {
            const response = await fetch(API_URL, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(reservation)
            });

            if (response.ok) {
                loadReservations();
                resetForm();
            } else {
                console.error('Server error:', await response.text());
            }
        } catch (error) {
            console.error('Error:', error);
        }
    }

    // Reset form
    function resetForm() {
        formData = {
            guestName: '',
            startDate: '',
            startTime: '',
            endDate: '',
            endTime: '',
            payment: 0
        };
    }

    // Load reservations
    async function loadReservations() {
        try {
            const response = await fetch(API_URL);
            if (response.ok) {
                const data = await response.json();
                updateRooms(data);
            } else {
                console.error('Server error:', await response.text());
            }
        } catch (error) {
            console.error('Error:', error);
        }
    }

    // Update rooms with reservation data
    function updateRooms(reservations: Reservation[]) {
        rooms = rooms.map(room => {
            const reservation = reservations.find(r => r.roomNumber === room.number);
            return {
                ...room,
                status: reservation ? 'occupied' : 'vacant',
                reservation: reservation || null
            };
        });
    }

    // Get current date in YYYY-MM-DD format
    const today = new Date().toISOString().split('T')[0];
    
    // Computed total amount that updates when dates change
    $: totalAmount = formData.startDate && formData.endDate ? 
        calculateAmount(`${formData.startDate}T${formData.startTime || '00:00'}`, 
                       `${formData.endDate}T${formData.endTime || '00:00'}`) : 0;

    // Validate end date is after start date
    $: isValidDateRange = formData.startDate && formData.endDate ? 
        new Date(`${formData.startDate}T${formData.startTime || '00:00'}`) < 
        new Date(`${formData.endDate}T${formData.endTime || '00:00'}`) : true;

    onMount(loadReservations);
</script>

<main>
    <div class="container">
        <header>
            <h1>Hotel Reservation System</h1>
            <p class="subtitle">Book your stay with ease</p>
        </header>

        <!-- Reservation Form -->
        <div class="card reservation-form">
            <h2>New Reservation</h2>
            <form on:submit|preventDefault={handleSubmit}>
                <div class="form-group">
                    <label for="guestName">Guest Name</label>
                    <input 
                        type="text" 
                        id="guestName"
                        placeholder="Enter full name"
                        bind:value={formData.guestName}
                        required
                    />
                </div>

                <div class="form-row">
                    <div class="form-group">
                        <label for="startDate">Check-in Date</label>
                        <input 
                            type="date" 
                            id="startDate"
                            bind:value={formData.startDate}
                            min={today}
                            required
                        />
                    </div>
                    <div class="form-group">
                        <label for="startTime">Check-in Time</label>
                        <input 
                            type="time" 
                            id="startTime"
                            bind:value={formData.startTime}
                            required
                        />
                    </div>
                </div>

                <div class="form-row">
                    <div class="form-group">
                        <label for="endDate">Check-out Date</label>
                        <input 
                            type="date" 
                            id="endDate"
                            bind:value={formData.endDate}
                            min={formData.startDate || today}
                            required
                        />
                    </div>
                    <div class="form-group">
                        <label for="endTime">Check-out Time</label>
                        <input 
                            type="time" 
                            id="endTime"
                            bind:value={formData.endTime}
                            required
                        />
                    </div>
                </div>

                <div class="price-info">
                    <div class="total-amount">
                        <span>Total Amount:</span>
                        <span class="amount">₱{totalAmount}</span>
                    </div>
                    <div class="form-group payment-input">
                        <label for="payment">Payment Amount</label>
                        <div class="input-with-currency">
                            <span class="currency">₱</span>
                            <input 
                                type="number" 
                                id="payment"
                                bind:value={formData.payment}
                                min="0"
                                max={totalAmount}
                                required
                            />
                        </div>
                    </div>
                </div>

                <button 
                    type="submit" 
                    class="submit-btn" 
                    disabled={!isValidDateRange}
                >
                    Make Reservation
                </button>
            </form>
        </div>

        <!-- Rooms Display -->
        <div class="rooms-section">
            <h2>Room Status</h2>
            <div class="rooms-grid">
                {#each rooms as room}
                    <div class="room-card" class:occupied={room.status === 'occupied'}>
                        <div class="room-header">
                            <h3>Room {room.number}</h3>
                            <span class="status-badge" class:occupied={room.status === 'occupied'}>
                                {room.status === 'vacant' ? 'Available' : 'Occupied'}
                            </span>
                        </div>
                        {#if room.status === 'occupied' && room.reservation}
                            <div class="reservation-details">
                                <div class="detail-item">
                                    <i class="fas fa-user"></i>
                                    <span>{room.reservation.guestName}</span>
                                </div>
                                <div class="detail-item">
                                    <i class="fas fa-calendar-check"></i>
                                    <span>{formatDateTime(room.reservation.startDate)}</span>
                                </div>
                                <div class="detail-item">
                                    <i class="fas fa-calendar-times"></i>
                                    <span>{formatDateTime(room.reservation.endDate)}</span>
                                </div>
                                <div class="detail-item">
                                    <i class="fas fa-money-bill-wave"></i>
                                    <span>₱{room.reservation.totalAmount}</span>
                                </div>
                                <div class="payment-status" 
                                     class:paid={room.reservation.payment >= room.reservation.totalAmount}>
                                    {room.reservation.payment >= room.reservation.totalAmount ? 'PAID' : 'PENDING PAYMENT'}
                                </div>
                            </div>
                        {/if}
                    </div>
                {/each}
            </div>
        </div>
    </div>
</main>

<style>
    @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap');
    @import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css');

    main {
        font-family: 'Poppins', sans-serif;
        background-color: #f5f7fa;
        min-height: 100vh;
        padding: 2rem;
    }

    .container {
        max-width: 1200px;
        margin: 0 auto;
    }

    header {
        text-align: center;
        margin-bottom: 3rem;
    }

    h1 {
        color: #1a237e;
        font-size: 2.5rem;
        margin-bottom: 0.5rem;
    }

    .subtitle {
        color: #666;
        font-size: 1.1rem;
    }

    .card {
        background: white;
        border-radius: 12px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        padding: 2rem;
        margin-bottom: 2rem;
    }

    .reservation-form h2 {
        color: #1a237e;
        margin-bottom: 1.5rem;
    }

    .form-group {
        margin-bottom: 1.5rem;
    }

    .form-row {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 1rem;
    }

    label {
        display: block;
        color: #444;
        margin-bottom: 0.5rem;
        font-weight: 500;
    }

    input {
        width: 100%;
        padding: 0.75rem;
        border: 1px solid #ddd;
        border-radius: 6px;
        font-size: 1rem;
        transition: border-color 0.3s;
    }

    input:focus {
        border-color: #1a237e;
        outline: none;
    }

    .price-info {
        background: #f8f9fa;
        padding: 1rem;
        border-radius: 8px;
        margin-bottom: 1.5rem;
    }

    .total-amount {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 1rem;
        font-weight: 600;
    }

    .amount {
        color: #1a237e;
        font-size: 1.25rem;
    }

    .input-with-currency {
        position: relative;
    }

    .currency {
        position: absolute;
        left: 1rem;
        top: 50%;
        transform: translateY(-50%);
        color: #666;
    }

    .payment-input input {
        padding-left: 2rem;
    }

    .submit-btn {
        background: #1a237e;
        color: white;
        border: none;
        padding: 1rem;
        border-radius: 6px;
        width: 100%;
        font-size: 1rem;
        font-weight: 500;
        cursor: pointer;
        transition: background-color 0.3s;
    }

    .submit-btn:hover:not(:disabled) {
        background: #283593;
    }

    .submit-btn:disabled {
        background: #ccc;
        cursor: not-allowed;
    }

    .rooms-section {
        margin-top: 3rem;
    }

    .rooms-section h2 {
        color: #1a237e;
        margin-bottom: 1.5rem;
    }

    .rooms-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
        gap: 1.5rem;
    }

    .room-card {
        background: white;
        border-radius: 12px;
        padding: 1.5rem;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        transition: transform 0.3s;
    }

    .room-card:hover {
        transform: translateY(-5px);
    }

    .room-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 1rem;
    }

    .room-header h3 {
        color: #1a237e;
        margin: 0;
    }

    .status-badge {
        padding: 0.5rem 1rem;
        border-radius: 20px;
        font-size: 0.875rem;
        font-weight: 500;
        background: #e8f5e9;
        color: #2e7d32;
    }

    .status-badge.occupied {
        background: #ffebee;
        color: #c62828;
    }

    .reservation-details {
        margin-top: 1rem;
        padding-top: 1rem;
        border-top: 1px solid #eee;
    }

    .detail-item {
        display: flex;
        align-items: center;
        margin-bottom: 0.75rem;
        color: #666;
    }

    .detail-item i {
        width: 20px;
        margin-right: 0.75rem;
        color: #1a237e;
    }

    .payment-status {
        margin-top: 1rem;
        padding: 0.5rem;
        text-align: center;
        border-radius: 4px;
        font-size: 0.875rem;
        font-weight: 500;
        background: #ffebee;
        color: #c62828;
    }

    .payment-status.paid {
        background: #e8f5e9;
        color: #2e7d32;
    }

    @media (max-width: 768px) {
        .form-row {
            grid-template-columns: 1fr;
        }

        .rooms-grid {
            grid-template-columns: 1fr;
        }

        main {
            padding: 1rem;
        }
    }
</style>
