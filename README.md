# LearnPress-Stats-Dashboard

<?php
/*
Plugin Name: LearnPress Stats Dashboard
Description: Hiển thị thống kê LearnPress
Version: 1.0
Author: Student
*/

if (!defined('ABSPATH')) {
    exit;
}

if (!class_exists('LP_Stats_Dashboard')) {

    class LP_Stats_Dashboard {
        const TRANSIENT_KEY = 'lp_stats_dashboard_data';
        const CACHE_TIME    = 300; // 5 phút

        public function __construct() {
            add_action('wp_dashboard_setup', [$this, 'register_dashboard_widget']);
            add_shortcode('lp_stats', [$this, 'shortcode']);
        }

        public function register_dashboard_widget() {
            wp_add_dashboard_widget(
                'lp_stats_widget',
                'LearnPress Statistics',
                [$this, 'render_dashboard_widget']
            );
        }

        public function get_data() {
            $cached = get_transient(self::TRANSIENT_KEY);
            if ($cached !== false) {
                return $cached;
            }

            global $wpdb;

            // Tổng số khóa học
            $total_courses = (int) $wpdb->get_var(
                "SELECT COUNT(ID)
                 FROM {$wpdb->posts}
                 WHERE post_type = 'lp_course'
                 AND post_status = 'publish'"
            );

            // Tổng số học viên (role subscriber)
            // Dùng count_users() nhẹ hơn get_users()
            $users_counts = count_users();
            $total_students = isset($users_counts['avail_roles']['subscriber'])
                ? (int) $users_counts['avail_roles']['subscriber']
                : 0;

            // Số khóa học đã hoàn thành
            $completed_courses = (int) $wpdb->get_var(
                "SELECT COUNT(*)
                 FROM {$wpdb->prefix}learnpress_user_items
                 WHERE status = 'completed'
                 AND item_type = 'lp_course'"
            );

            $data = [
                'courses'   => $total_courses,
                'students'  => $total_students,
                'completed' => $completed_courses,
            ];

            set_transient(self::TRANSIENT_KEY, $data, self::CACHE_TIME);

            return $data;
        }

        public function render_dashboard_widget() {
            $data = $this->get_data();

            echo '<ul style="margin:0; padding-left:18px;">';
            echo '<li><strong>Tổng khóa học:</strong> ' . esc_html($data['courses']) . '</li>';
            echo '<li><strong>Tổng học viên:</strong> ' . esc_html($data['students']) . '</li>';
            echo '<li><strong>Khóa học đã hoàn thành:</strong> ' . esc_html($data['completed']) . '</li>';
            echo '</ul>';
        }

        public function shortcode() {
            $data = $this->get_data();

            ob_start();
            ?>
            <div style="border:1px solid #ccc; padding:16px; border-radius:8px;">
                <h3 style="margin-top:0;">Thống kê LearnPress</h3>
                <p><strong>Tổng khóa học:</strong> <?php echo esc_html($data['courses']); ?></p>
                <p><strong>Tổng học viên:</strong> <?php echo esc_html($data['students']); ?></p>
                <p><strong>Khóa học đã hoàn thành:</strong> <?php echo esc_html($data['completed']); ?></p>
            </div>
            <?php
            return ob_get_clean();
        }
    }

    new LP_Stats_Dashboard();
}
