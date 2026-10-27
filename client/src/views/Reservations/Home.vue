<template>
    <div class="route-container">
        <HeaderBar />
        <main>
            <Navigation />
            <div class="reservations-home-container-wrapper">
                <SPHeaderBar />
                <div class="reservations-home-container">
                    <Breadcrumbs :crumbs="[
                        { name: 'ホーム', path: '/' },
                        { name: '録画予約', path: '/reservations/', disabled: true },
                    ]" />
                    <!-- 放送が近い録画予約セクション -->
                    <ReservationList
                        class="reservations-home-container__upcoming-reservations"
                        :class="{'reservations-home-container__upcoming-reservations--loading': upcomingReservations.length === 0 && isLoading}"
                        title="放送が近い録画予約"
                        :reservations="upcomingReservations"
                        :total="totalUpcomingReservations"
                        :hideSort="true"
                        :hidePagination="true"
                        :showMoreButton="true"
                        :isLoading="isLoading"
                        :showEmptyMessage="!isLoading"
                        :emptyIcon="'fluent:timer-20-regular'"
                        :emptyMessage="'まだ録画予約がありません。'"
                        :emptySubMessage="'番組表から録画予約を追加できます。'"
                        @more="$router.push('/reservations/all')"
                        @delete="handleReservationDeleted" />
                    <!-- キーワード自動予約への導線 -->
                    <div v-ripple class="reservations-home-container__conditions-link"
                        @click="$router.push('/reservations/conditions')">
                        <Icon icon="fluent:text-bullet-list-square-search-20-regular" width="26px" height="26px"
                            class="reservations-home-container__conditions-link-icon" />
                        <div class="reservations-home-container__conditions-link-text">
                            <div class="reservations-home-container__conditions-link-title">キーワード自動予約</div>
                            <div class="reservations-home-container__conditions-link-description">
                                番組名などのキーワードで、毎週の番組や再放送を自動的に録画予約します。
                            </div>
                        </div>
                        <Icon icon="fluent:chevron-right-12-regular" width="20px" height="20px" class="text-text-darken-1" />
                    </div>
                </div>
            </div>
        </main>
    </div>
</template>
<script lang="ts" setup>

import { onMounted, ref, onUnmounted } from 'vue';

import Breadcrumbs from '@/components/Breadcrumbs.vue';
import HeaderBar from '@/components/HeaderBar.vue';
import Navigation from '@/components/Navigation.vue';
import ReservationList from '@/components/Reservations/ReservationList.vue';
import SPHeaderBar from '@/components/SPHeaderBar.vue';
import Reservations, { IReservation } from '@/services/Reservations';
import { dayjs } from '@/utils';

// 放送が近い録画予約のリスト
const upcomingReservations = ref<IReservation[]>([]);
const totalUpcomingReservations = ref(0);

const isLoading = ref(true);

// 自動更新用の interval ID を保持
const autoRefreshInterval = ref<number | null>(null);

// 自動更新の間隔 (ミリ秒)
const AUTO_REFRESH_INTERVAL = 30 * 1000;  // 30秒

// 表示上限
const UPCOMING_DISPLAY_LIMIT = 10;

// 放送が近い録画予約を取得
const fetchUpcomingReservations = async () => {
    const result = await Reservations.fetchReservations();
    if (result) {
        const now = dayjs();
        const allFetchedReservations = result.reservations;

        // 放送が近い録画予約（24時間以内に開始）および録画中の番組を取得
        const upcomingOrRecording = allFetchedReservations
            .filter(res => {
                const startTime = dayjs(res.program.start_time);
                const isUpcoming = startTime.isAfter(now) && startTime.isBefore(now.add(24, 'hours'));
                const isRecording = res.is_recording_in_progress;
                return isUpcoming || isRecording;
            })
            .sort((a, b) => {
                // 録画中を優先、その後は開始時間順
                if (a.is_recording_in_progress && !b.is_recording_in_progress) return -1;
                if (!a.is_recording_in_progress && b.is_recording_in_progress) return 1;
                return dayjs(a.program.start_time).valueOf() - dayjs(b.program.start_time).valueOf();
            });

        totalUpcomingReservations.value = upcomingOrRecording.length;
        upcomingReservations.value = upcomingOrRecording.slice(0, UPCOMING_DISPLAY_LIMIT);
    }
};

// セクションの更新関数を管理するオブジェクト
const sectionUpdaters = {
    upcomingReservations: fetchUpcomingReservations,
} as const;

// 全セクションの更新を実行
const updateAllSections = async () => {
    try {
        // 全セクションの更新関数を実行
        await Promise.all(Object.values(sectionUpdaters).map(updater => updater()));
        isLoading.value = false;
    } catch (error) {
        console.error('Failed to update reservation sections:', error);
        isLoading.value = false;
    }
};

// 録画予約が削除された時の処理
const handleReservationDeleted = (deleted_reservation_id: number) => {
    // 削除イベントを受けたらリストを即時更新
    console.log(`Reservation ${deleted_reservation_id} deleted, refreshing list...`);
    updateAllSections();
};

// 自動更新を開始
const startAutoRefresh = () => {
    if (autoRefreshInterval.value === null) {
        // 初回更新
        updateAllSections();
        // 定期更新を開始
        autoRefreshInterval.value = window.setInterval(updateAllSections, AUTO_REFRESH_INTERVAL);
    }
};

// 自動更新を停止
const stopAutoRefresh = () => {
    if (autoRefreshInterval.value !== null) {
        clearInterval(autoRefreshInterval.value);
        autoRefreshInterval.value = null;
    }
};

// 開始時に実行
onMounted(() => {
    startAutoRefresh();
});

// コンポーネントのクリーンアップ
onUnmounted(() => {
    stopAutoRefresh();
});

</script>
<style lang="scss" scoped>

.reservations-home-container-wrapper {
    display: flex;
    flex-direction: column;
    width: 100%;
    min-width: 0;  // very important!!! これがないと要素がはみ出す
}

.reservations-home-container {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;
    padding: 20px;
    margin: 0 auto;
    min-width: 0;
    max-width: 1000px;

    @include smartphone-horizontal {
        padding: 16px 20px !important;
    }

    @include smartphone-horizontal-short {
        padding: 16px !important;
    }

    @include smartphone-vertical {
        padding: 8px 8px 20px !important;
    }



    :deep(.reservation-list) {
        & + .reservation-list {
            margin-top: 40px;
            @include smartphone-vertical {
                margin-top: 32px;
            }
        }
    }




    &__conditions-link {
        display: flex;
        align-items: center;
        gap: 14px;
        margin-top: 32px;
        padding: 14px 16px;
        border-radius: 12px;
        background: rgb(var(--v-theme-background-lighten-1));
        cursor: pointer;
        user-select: none;
        @include smartphone-vertical {
            margin-top: 24px;
            padding: 12px 14px;
        }
    }
    &__conditions-link-icon {
        flex-shrink: 0;
        color: rgb(var(--v-theme-secondary));
    }
    &__conditions-link-text {
        flex: 1;
        min-width: 0;
    }
    &__conditions-link-title {
        font-size: 16px;
        font-weight: 700;
    }
    &__conditions-link-description {
        margin-top: 2px;
        font-size: 12.5px;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__upcoming-reservations--loading,
    &__recently-finished-reservations--loading,
    &__all-reservations--loading {
        :deep(.reservation-list__table-container),
        :deep(.reservation-list__empty) {
            min-height: 180px;
        }
    }
}
</style>
