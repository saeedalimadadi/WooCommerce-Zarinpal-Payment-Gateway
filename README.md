# WooCommerce Zarinpal Payment Gateway

This is a custom **WooCommerce payment gateway integration for Zarinpal**, one of the most popular Iranian online payment services. This code allows you to add Zarinpal as a payment option on your WooCommerce checkout page, enabling your customers to pay for their orders securely through the Zarinpal payment portal.

## Why use this code?

* **Accept Payments via Zarinpal:** Seamlessly integrate Zarinpal as a payment method for your WooCommerce store.
* **Secure Transactions:** Facilitates secure redirection to the Zarinpal gateway for payment processing.
* **Admin Configuration:** Provides an intuitive settings page within WooCommerce to configure your Zarinpal Merchant ID.
* **Automated Order Status Updates:** Automatically updates order statuses based on payment success or failure.
* **Comprehensive Error Handling:** Includes detailed error handling with user-friendly messages for various transaction outcomes from Zarinpal's API.
* **Currency Conversion:** Automatically converts Toman/IRT to Rial for Zarinpal's API if your store's currency is set to Toman or IRR (Zarinpal's API typically expects amounts in Rial).

## Features

* Adds "Zarinpal" as a selectable payment gateway in WooCommerce settings.
* Allows configuration of gateway title, description, and Merchant ID.
* Handles the entire payment flow:
    * Initiates payment request to Zarinpal.
    * Redirects customer to Zarinpal payment page.
    * Manages the callback (return from Zarinpal).
    * Verifies payment with Zarinpal's API.
    * Completes WooCommerce order for successful payments.
    * Updates order status and adds notes for successful, cancelled, or failed payments.
* Translates Zarinpal API error codes into human-readable messages.
* Sends customer's email and mobile as metadata to Zarinpal, if available.

## Installation

This code functions as a custom WooCommerce payment gateway plugin.

1.  **Create Plugin Folder:** In your WordPress installation, navigate to `wp-content/plugins/` and create a new folder named `woocommerce-zarinpal-gateway`.
2.  **Create Plugin File:** Inside the `woocommerce-zarinpal-gateway` folder, create a file named `woocommerce-zarinpal-gateway.php`.
3.  **Paste Code:** Copy and paste the entire code provided into the `woocommerce-zarinpal-gateway.php` file.

    ```php
    <?php
    /**
     * Plugin Name: WooCommerce Zarinpal Payment Gateway
     * Description: Integrates Zarinpal as a payment gateway for WooCommerce.
     * Version: 1.0
     * Author: Your Name (Optional)
     * License: GPL-2.0-or-later
     * Text Domain: wc-zarinpal-gateway
     *
     * @package WooCommerce
     * @subpackage Zarinpal_Gateway
     * @author Masoud Aroodipoor
     */

    if ( ! defined( 'ABSPATH' ) ) {
        exit; // Exit if accessed directly.
    }

    /**
     * Adds the Zarinpal gateway class to WooCommerce.
     *
     * @param array $gateways Existing WooCommerce payment gateways.
     * @return array Modified list of gateways.
     */
    add_filter('woocommerce_payment_gateways', 'add_zarinpal_gateway_to_wc');
    function add_zarinpal_gateway_to_wc($gateways)
    {
        $gateways[] = 'WC_Zarinpal_Gateway';
        return $gateways;
    }

    /**
     * Initializes the Zarinpal payment gateway class.
     * Ensures WC_Payment_Gateway class is available before proceeding.
     */
    add_action('plugins_loaded', 'init_zarinpal_gateway');
    function init_zarinpal_gateway()
    {
        if (!class_exists('WC_Payment_Gateway')) {
            return;
        }

        class WC_Zarinpal_Gateway extends WC_Payment_Gateway
        {
            /**
             * @var string Merchant ID
             */
            public $merchant_id;

            /**
             * @var string Callback URL
             */
            public $callback_url;

            /**
             * Constructor for the gateway.
             */
            public function __construct()
            {
                $this->id = 'zarinpal';
                $this->method_title = 'Zarinpal'; // Displayed in admin
                $this->method_description = 'Secure payment through Zarinpal payment gateway.'; // Displayed in admin
                $this->has_fields = false; // No extra fields on checkout
                $this->icon = apply_filters('woocommerce_zarinpal_icon', plugin_dir_url(__FILE__) . 'zarinpal.png'); // Path to gateway icon

                // Initialize settings and form fields
                $this->init_form_fields();
                $this->init_settings();

                // Get values from settings
                $this->title = $this->get_option('title'); // Displayed on checkout page
                $this->description = $this->get_option('description'); // Displayed on checkout page
                $this->merchant_id = $this->get_option('merchant_id');
                // WooCommerce API callback URL: [yoursite.com/?wc-api=zarinpal_callback](https://yoursite.com/?wc-api=zarinpal_callback)
                $this->callback_url = WC()->api_request_url('zarinpal_callback');

                // Register hooks
                add_action('woocommerce_update_options_payment_gateways_' . $this->id, [$this, 'process_admin_options']);
                add_action('woocommerce_api_zarinpal_callback', [$this, 'handle_callback']); // Handles the return from Zarinpal
            }

            /**
             * Define gateway settings fields.
             */
            public function init_form_fields()
            {
                $this->form_fields = [
                    'enabled' => [
                        'title'   => __('Enable/Disable', 'wc-zarinpal-gateway'),
                        'type'    => 'checkbox',
                        'label'   => __('Enable Zarinpal Gateway', 'wc-zarinpal-gateway'),
                        'default' => 'yes',
                    ],
                    'title' => [
                        'title'       => __('Title', 'wc-zarinpal-gateway'),
                        'type'        => 'text',
                        'description' => __('This controls the title which the user sees during checkout.', 'wc-zarinpal-gateway'),
                        'default'     => __('Pay with Zarinpal', 'wc-zarinpal-gateway'),
                        'desc_tip'    => true,
                    ],
                    'description' => [
                        'title'       => __('Description', 'wc-zarinpal-gateway'),
                        'type'        => 'textarea',
                        'description' => __('This controls the description which the user sees during checkout.', 'wc-zarinpal-gateway'),
                        'default'     => __('Secure payment through Zarinpal gateway.', 'wc-zarinpal-gateway'),
                    ],
                    'merchant_id' => [
                        'title'       => __('Merchant ID', 'wc-zarinpal-gateway'),
                        'type'        => 'text',
                        'description' => __('Enter your Zarinpal Merchant ID. You can get this from your Zarinpal panel.', 'wc-zarinpal-gateway'),
                    ],
                ];
            }

            /**
             * Process the payment.
             *
             * @param int $order_id The ID of the order.
             * @return array An array of result and redirect URL.
             */
            public function process_payment($order_id)
            {
                $order = wc_get_order($order_id);
                $amount = (int) $order->get_total();
                
                // Convert Toman to Rial if currency is Toman/IRT
                if (strtolower(get_woocommerce_currency()) === 'irt' || strtolower(get_woocommerce_currency()) === 'toman') {
                    $amount *= 10;
                }

                $data = [
                    'merchant_id'  => $this->merchant_id,
                    'amount'       => $amount,
                    'description'  => 'Payment for order: ' . $order->get_order_number(),
                    'callback_url' => add_query_arg('order_id', $order_id, $this->callback_url),
                    'metadata'     => [
                        'email'  => $order->get_billing_email(),
                        'mobile' => $order->get_billing_phone(),
                    ],
                ];

                $response = $this->send_request_to_api('[https://api.zarinpal.com/pg/v4/payment/request.json](https://api.zarinpal.com/pg/v4/payment/request.json)', $data);

                if (isset($response['data']['authority'])) {
                    $authority = $response['data']['authority'];
                    $redirect_url = '[https://www.zarinpal.com/pg/StartPay/](https://www.zarinpal.com/pg/StartPay/)' . $authority;
                    
                    return [
                        'result'   => 'success',
                        'redirect' => $redirect_url,
                    ];
                } else {
                    $error_code = $response['errors']['code'] ?? 'unknown';
                    $error_message = $this->get_error_message($error_code);
                    wc_add_notice(__('Error connecting to payment gateway:', 'wc-zarinpal-gateway') . ' ' . $error_message, 'error');
                    return ['result' => 'failure'];
                }
            }

            /**
             * Handle the callback from Zarinpal gateway.
             */
            public function handle_callback()
            {
                if (empty($_GET['Authority']) || empty($_GET['Status']) || empty($_GET['order_id'])) {
                    wc_add_notice(__('Incomplete return information from the gateway.', 'wc-zarinpal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                $order_id  = absint($_GET['order_id']);
                $authority = sanitize_text_field($_GET['Authority']);
                $status    = sanitize_text_field($_GET['Status']); // "OK" for success
                $order     = wc_get_order($order_id);

                if (!$order || !$order->get_id()) {
                    wc_add_notice(__('Order not found.', 'wc-zarinpal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                if ($status === 'OK') {
                    $amount = (int) $order->get_total();
                    if (strtolower(get_woocommerce_currency()) === 'irt' || strtolower(get_woocommerce_currency()) === 'toman') {
                        $amount *= 10;
                    }
                    
                    $data = [
                        'merchant_id' => $this->merchant_id,
                        'authority'   => $authority,
                        'amount'      => $amount,
                    ];

                    $response = $this->send_request_to_api('[https://api.zarinpal.com/pg/v4/payment/verify.json](https://api.zarinpal.com/pg/v4/payment/verify.json)', $data);

                    if (isset($response['data']['code']) && $response['data']['code'] == 100) {
                        $ref_id = $response['data']['ref_id'];
                        $order->payment_complete($ref_id);
                        $order->add_order_note(__('Payment successful. Zarinpal Tracking ID:', 'wc-zarinpal-gateway') . ' ' . $ref_id);
                        wc_add_notice(__('Your payment was successful.', 'wc-zarinpal-gateway'), 'success');
                        wp_redirect($this->get_return_url($order));
                        exit;
                    } else {
                        $error_code = $response['errors']['code'] ?? 'unknown';
                        $error_message = $this->get_error_message($error_code);
                        $order->update_status('failed', __('Payment failed:', 'wc-zarinpal-gateway') . ' ' . $error_message);
                        wc_add_notice(__('Error in transaction verification:', 'wc-zarinpal-gateway') . ' ' . $error_message, 'error');
                    }

                } else {
                    $order->update_status('cancelled', __('Payment cancelled by user.', 'wc-zarinpal-gateway'));
                    wc_add_notice(__('Payment was cancelled by you.', 'wc-zarinpal-gateway'), 'error');
                }

                wp_redirect(wc_get_checkout_url());
                exit;
            }

            /**
             * Sends a request to the Zarinpal API.
             *
             * @param string $url  API endpoint URL.
             * @param array  $data Data to send in the request body.
             * @return array Decoded JSON response from the API.
             */
            private function send_request_to_api($url, $data) {
                $response = wp_remote_post($url, [
                    'method'  => 'POST',
                    'headers' => ['Content-Type' => 'application/json', 'Accept' => 'application/json'],
                    'body'    => json_encode($data),
                    'timeout' => 30, // seconds
                ]);

                if (is_wp_error($response)) {
                    error_log('Zarinpal API Connection Error: ' . $response->get_error_message());
                    return ['errors' => ['code' => -999, 'message' => 'Connection Error']];
                }
                
                return json_decode(wp_remote_retrieve_body($response), true);
            }

            /**
             * Translates Zarinpal error codes into human-readable messages.
             *
             * @param int|string $code The Zarinpal error code.
             * @return string Translated error message.
             */
            private function get_error_message($code) {
                $errors = [
                    '-1'      => __('Information submitted is incomplete.', 'wc-zarinpal-gateway'),
                    '-2'      => __('Receiver\'s IP or Merchant ID is incorrect.', 'wc-zarinpal-gateway'),
                    '-3'      => __('Payment with the requested amount is not possible due to Shaparak restrictions.', 'wc-zarinpal-gateway'),
                    '-4'      => __('Receiver\'s verification level is lower than Silver.', 'wc-zarinpal-gateway'),
                    '-11'     => __('The requested transaction was not found.', 'wc-zarinpal-gateway'),
                    '-21'     => __('No financial operation found for this transaction.', 'wc-zarinpal-gateway'),
                    '-22'     => __('Transaction is unsuccessful.', 'wc-zarinpal-gateway'),
                    '-33'     => __('Transaction amount does not match the paid amount.', 'wc-zarinpal-gateway'),
                    '-34'     => __('Transaction splitting limit exceeded (by count or amount).', 'wc-zarinpal-gateway'),
                    '-40'     => __('Access to the related method is not allowed.', 'wc-zarinpal-gateway'),
                    '-41'     => __('Submitted information related to AdditionalData is invalid.', 'wc-zarinpal-gateway'),
                    '-42'     => __('Payment ID validity period must be between 30 minutes and 45 days.', 'wc-zarinpal-gateway'),
                    '-54'     => __('The requested transaction has been archived.', 'wc-zarinpal-gateway'),
                    '100'     => __('Operation was successful.', 'wc-zarinpal-gateway'),
                    '101'     => __('Payment operation was successful and transaction verification has already been done.', 'wc-zarinpal-gateway'),
                    '-999'    => __('Error connecting to Zarinpal server.', 'wc-zarinpal-gateway'),
                    'unknown' => __('An unknown error occurred.', 'wc-zarinpal-gateway'),
                ];

                return $errors[$code] ?? sprintf(__('Undefined error with code: %s', 'wc-zarinpal-gateway'), $code);
            }
        }
    }
    ```

4.  **Upload Icon (Optional but Recommended):** Place your Zarinpal payment gateway icon (e.g., `zarinpal.png`) inside the `woocommerce-zarinpal-gateway` folder, next to `woocommerce-zarinpal-gateway.php`. This icon will appear next to the payment gateway name on the checkout page.
5.  **Activate Plugin:** Go to your WordPress admin dashboard, navigate to **"Plugins"**, and **activate** the "WooCommerce Zarinpal Payment Gateway" plugin.
6.  **Configure Gateway:** Go to **WooCommerce > Settings > Payments**. You should now see "Zarinpal" listed. Click on "Manage" (or "Set up") to configure your **Merchant ID** and other settings.

## Important Considerations

* **Merchant ID:** You **MUST** obtain a valid Merchant ID from Zarinpal to use this gateway.
* **Currency:** Zarinpal's API expects the amount in **Rials**. The provided code includes logic to convert Toman/IRT to Rial (`$amount *= 10`). Ensure your WooCommerce currency is correctly set up, and this conversion matches Zarinpal's current API expectations.
* **SSL Certificate:** For security, ensure your website has a valid SSL certificate (HTTPS) installed. Payment gateways require secure connections.
* **Error Logging:** The `error_log` calls are useful for debugging connection issues. Check your server's PHP error logs if you encounter problems.
* **Security:** This is a basic implementation. For production environments, always consider additional security best practices, such as IP whitelisting if supported by Zarinpal, and regular security audits.
* **Updates:** Custom gateway code needs to be maintained. Ensure you stay updated with any changes in Zarinpal's API or WooCommerce's payment gateway standards.

## Contributing

Contributions are welcome! If you have suggestions or improvements for this code, feel free to open a "Pull Request" or report an "Issue."

## License

This project is licensed under the GPL-2.0-or-later License.

---


# درگاه پرداخت زرین پال برای ووکامرس

این کد یکپارچه‌سازی سفارشی **درگاه پرداخت زرین پال برای ووکامرس** است. زرین پال یکی از محبوب‌ترین سرویس‌های پرداخت آنلاین ایرانی است. این کد به شما امکان می‌دهد زرین پال را به عنوان یک گزینه پرداخت در صفحه تسویه‌حساب ووکامرس خود اضافه کنید و مشتریان خود را قادر می‌سازد تا سفارشات خود را به طور امن از طریق پورتال پرداخت زرین پال پرداخت کنند.

## چرا از این کد استفاده کنیم؟

* **پذیرش پرداخت از طریق زرین پال:** زرین پال را به طور یکپارچه به عنوان یک روش پرداخت برای فروشگاه ووکامرس خود اضافه کنید.
* **تراکنش‌های امن:** انتقال امن به درگاه زرین پال برای پردازش پرداخت را تسهیل می‌کند.
* **پیکربندی مدیر:** یک صفحه تنظیمات بصری در ووکامرس برای پیکربندی مرچنت کد زرین پال شما فراهم می‌کند.
* **به‌روزرسانی خودکار وضعیت سفارش:** وضعیت سفارش را به طور خودکار بر اساس موفقیت یا عدم موفقیت پرداخت به‌روزرسانی می‌کند.
* **مدیریت خطای جامع:** شامل مدیریت خطای دقیق با پیام‌های کاربرپسند برای نتایج مختلف تراکنش‌ها از API زرین پال است.
* **تبدیل ارز:** اگر ارز فروشگاه شما تومان یا IRR باشد، به طور خودکار تومان/IRT را به ریال برای API زرین پال تبدیل می‌کند (API زرین پال معمولاً مبالغ را به ریال انتظار دارد).

## قابلیت‌ها

* "زرین پال" را به عنوان یک درگاه پرداخت قابل انتخاب در تنظیمات ووکامرس اضافه می‌کند.
* امکان پیکربندی عنوان درگاه، توضیحات و مرچنت کد را فراهم می‌کند.
* تمام جریان پرداخت را مدیریت می‌کند:
    * آغاز درخواست پرداخت به زرین پال.
    * هدایت مشتری به صفحه پرداخت زرین پال.
    * مدیریت بازگشت از زرین پال (callback).
    * تأیید پرداخت با API زرین پال.
    * تکمیل سفارش ووکامرس برای پرداخت‌های موفق.
    * به‌روزرسانی وضعیت سفارش و افزودن یادداشت‌ها برای پرداخت‌های موفق، لغو شده یا ناموفق.
* کدهای خطای API زرین پال را به پیام‌های قابل فهم برای انسان ترجمه می‌کند.
* ایمیل و شماره موبایل مشتری را در صورت موجود بودن، به عنوان متاداده به زرین پال ارسال می‌کند.

## نصب

این کد به عنوان یک افزونه درگاه پرداخت سفارشی ووکامرس عمل می‌کند.

1.  **ایجاد پوشه افزونه:** در نصب وردپرس خود، به مسیر `wp-content/plugins/` بروید و یک پوشه جدید به نام `woocommerce-zarinpal-gateway` ایجاد کنید.
2.  **ایجاد فایل افزونه:** در داخل پوشه `woocommerce-zarinpal-gateway`، یک فایل به نام `woocommerce-zarinpal-gateway.php` ایجاد کنید.
3.  **جایگذاری کد:** کل کد ارائه شده را کپی کرده و در فایل `woocommerce-zarinpal-gateway.php` جایگذاری کنید.

    ```php
    <?php
    /**
     * Plugin Name: WooCommerce Zarinpal Payment Gateway
     * Description: Integrates Zarinpal as a payment gateway for WooCommerce.
     * Version: 1.0
     * Author: Your Name (Optional)
     * License: GPL-2.0-or-later
     * Text Domain: wc-zarinpal-gateway
     *
     * @package WooCommerce
     * @subpackage Zarinpal_Gateway
     * @author Masoud Aroodipoor
     */

    if ( ! defined( 'ABSPATH' ) ) {
        exit; // Exit if accessed directly.
    }

    /**
     * افزودن درگاه زرین پال به لیست درگاه‌های ووکامرس
     *
     * @param array $gateways درگاه‌های پرداخت موجود ووکامرس.
     * @return array لیست ویرایش شده درگاه‌ها.
     */
    add_filter('woocommerce_payment_gateways', 'add_zarinpal_gateway_to_wc');
    function add_zarinpal_gateway_to_wc($gateways)
    {
        $gateways[] = 'WC_Zarinpal_Gateway';
        return $gateways;
    }

    /**
     * مقداردهی اولیه کلاس درگاه پرداخت
     * اطمینان حاصل می‌کند که کلاس WC_Payment_Gateway قبل از ادامه در دسترس باشد.
     */
    add_action('plugins_loaded', 'init_zarinpal_gateway');
    function init_zarinpal_gateway()
    {
        if (!class_exists('WC_Payment_Gateway')) {
            return;
        }

        class WC_Zarinpal_Gateway extends WC_Payment_Gateway
        {
            /**
             * @var string کد مرچنت
             */
            public $merchant_id;

            /**
             * @var string آدرس بازگشت از درگاه
             */
            public $callback_url;

            /**
             * سازنده کلاس درگاه.
             */
            public function __construct()
            {
                $this->id = 'zarinpal';
                $this->method_title = 'زرین پال'; // نمایش در مدیریت
                $this->method_description = 'پرداخت امن از طریق درگاه پرداخت زرین پال'; // نمایش در مدیریت
                $this->has_fields = false; // بدون فیلدهای اضافی در تسویه‌حساب
                $this->icon = apply_filters('woocommerce_zarinpal_icon', plugin_dir_url(__FILE__) . 'zarinpal.png'); // مسیر آیکون درگاه

                // مقداردهی اولیه تنظیمات و فیلدهای فرم
                $this->init_form_fields();
                $this->init_settings();

                // دریافت مقادیر از تنظیمات
                $this->title = $this->get_option('title'); // نمایش در صفحه تسویه‌حساب
                $this->description = $this->get_option('description'); // نمایش در صفحه تسویه‌حساب
                $this->merchant_id = $this->get_option('merchant_id');
                // آدرس بازگشت API ووکامرس: yoursite.com/?wc-api=zarinpal_callback
                $this->callback_url = WC()->api_request_url('zarinpal_callback');

                // ثبت هوک‌ها
                add_action('woocommerce_update_options_payment_gateways_' . $this->id, [$this, 'process_admin_options']);
                add_action('woocommerce_api_zarinpal_callback', [$this, 'handle_callback']); // بازگشت از زرین پال را مدیریت می‌کند
            }

            /**
             * تعریف فیلدهای تنظیمات درگاه.
             */
            public function init_form_fields()
            {
                $this->form_fields = [
                    'enabled' => [
                        'title'   => __('فعال‌سازی/غیرفعال‌سازی', 'wc-zarinpal-gateway'),
                        'type'    => 'checkbox',
                        'label'   => __('فعال کردن درگاه زرین پال', 'wc-zarinpal-gateway'),
                        'default' => 'yes',
                    ],
                    'title' => [
                        'title'       => __('عنوان', 'wc-zarinpal-gateway'),
                        'type'        => 'text',
                        'description' => __('این فیلد عنوانی است که مشتری در صفحه تسویه حساب مشاهده می‌کند.', 'wc-zarinpal-gateway'),
                        'default'     => __('پرداخت با زرین پال', 'wc-zarinpal-gateway'),
                        'desc_tip'    => true,
                    ],
                    'description' => [
                        'title'       => __('توضیحات', 'wc-zarinpal-gateway'),
                        'type'        => 'textarea',
                        'description' => __('این فیلد توضیحاتی است که در صفحه تسویه حساب نمایش داده می‌شود.', 'wc-zarinpal-gateway'),
                        'default'     => __('پرداخت امن از طریق درگاه زرین پال.', 'wc-zarinpal-gateway'),
                    ],
                    'merchant_id' => [
                        'title'       => __('مرچنت کد (Merchant ID)', 'wc-zarinpal-gateway'),
                        'type'        => 'text',
                        'description' => __('کد مرچنت که از زرین پال دریافت کرده‌اید.', 'wc-zarinpal-gateway'),
                    ],
                ];
            }

            /**
             * پردازش پرداخت.
             *
             * @param int $order_id شناسه سفارش.
             * @return array آرایه‌ای از نتیجه و URL هدایت.
             */
            public function process_payment($order_id)
            {
                $order = wc_get_order($order_id);
                $amount = (int) $order->get_total();
                
                // اگر واحد پول تومان بود، به ریال تبدیل کن
                if (strtolower(get_woocommerce_currency()) === 'irt' || strtolower(get_woocommerce_currency()) === 'toman') {
                    $amount *= 10;
                }

                $data = [
                    'merchant_id'  => $this->merchant_id,
                    'amount'       => $amount,
                    'description'  => 'پرداخت سفارش شماره: ' . $order->get_order_number(),
                    'callback_url' => add_query_arg('order_id', $order_id, $this->callback_url),
                    'metadata'     => [
                        'email'  => $order->get_billing_email(),
                        'mobile' => $order->get_billing_phone(),
                    ],
                ];

                $response = $this->send_request_to_api('https://api.zarinpal.com/pg/v4/payment/request.json', $data);

                if (isset($response['data']['authority'])) {
                    $authority = $response['data']['authority'];
                    $redirect_url = 'https://www.zarinpal.com/pg/StartPay/' . $authority;
                    
                    return [
                        'result'   => 'success',
                        'redirect' => $redirect_url,
                    ];
                } else {
                    $error_code = $response['errors']['code'] ?? 'unknown';
                    $error_message = $this->get_error_message($error_code);
                    wc_add_notice(__('خطا در اتصال به درگاه پرداخت:', 'wc-zarinpal-gateway') . ' ' . $error_message, 'error');
                    return ['result' => 'failure'];
                }
            }

            /**
             * مدیریت بازگشت از درگاه زرین‌پال.
             */
            public function handle_callback()
            {
                if (empty($_GET['Authority']) || empty($_GET['Status']) || empty($_GET['order_id'])) {
                    wc_add_notice(__('اطلاعات بازگشتی از درگاه ناقص است.', 'wc-zarinpal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                $order_id  = absint($_GET['order_id']);
                $authority = sanitize_text_field($_GET['Authority']);
                $status    = sanitize_text_field($_GET['Status']); // "OK" برای موفقیت
                $order     = wc_get_order($order_id);

                if (!$order || !$order->get_id()) {
                    wc_add_notice(__('سفارش یافت نشد.', 'wc-zarinpal-gateway'), 'error');
                    wp_redirect(wc_get_checkout_url());
                    exit;
                }

                if ($status === 'OK') {
                    $amount = (int) $order->get_total();
                    if (strtolower(get_woocommerce_currency()) === 'irt' || strtolower(get_woocommerce_currency()) === 'toman') {
                        $amount *= 10;
                    }
                    
                    $data = [
                        'merchant_id' => $this->merchant_id,
                        'authority'   => $authority,
                        'amount'      => $amount,
                    ];

                    $response = $this->send_request_to_api('https://api.zarinpal.com/pg/v4/payment/verify.json', $data);

                    if (isset($response['data']['code']) && $response['data']['code'] == 100) {
                        $ref_id = $response['data']['ref_id'];
                        $order->payment_complete($ref_id);
                        $order->add_order_note(__('پرداخت موفق. کد رهگیری زرین‌پال:', 'wc-zarinpal-gateway') . ' ' . $ref_id);
                        wc_add_notice(__('پرداخت شما با موفقیت انجام شد.', 'wc-zarinpal-gateway'), 'success');
                        wp_redirect($this->get_return_url($order));
                        exit;
                    } else {
                        $error_code = $response['errors']['code'] ?? 'unknown';
                        $error_message = $this->get_error_message($error_code);
                        $order->update_status('failed', __('پرداخت ناموفق:', 'wc-zarinpal-gateway') . ' ' . $error_message);
                        wc_add_notice(__('خطا در تایید تراکنش:', 'wc-zarinpal-gateway') . ' ' . $error_message, 'error');
                    }

                } else {
                    $order->update_status('cancelled', __('پرداخت توسط کاربر لغو شد.', 'wc-zarinpal-gateway'));
                    wc_add_notice(__('پرداخت توسط شما لغو شد.', 'wc-zarinpal-gateway'), 'error');
                }

                wp_redirect(wc_get_checkout_url());
                exit;
            }

            /**
             * ارسال درخواست به API زرین‌پال.
             *
             * @param string $url  آدرس نقطه پایانی API.
             * @param array  $data داده برای ارسال در بدنه درخواست.
             * @return array پاسخ JSON رمزگشایی شده از API.
             */
            private function send_request_to_api($url, $data) {
                $response = wp_remote_post($url, [
                    'method'  => 'POST',
                    'headers' => ['Content-Type' => 'application/json', 'Accept' => 'application/json'],
                    'body'    => json_encode($data),
                    'timeout' => 30, // ثانیه
                ]);

                if (is_wp_error($response)) {
                    error_log('Zarinpal API Connection Error: ' . $response->get_error_message());
                    return ['errors' => ['code' => -999, 'message' => 'Connection Error']];
                }
                
                return json_decode(wp_remote_retrieve_body($response), true);
            }

            /**
             * ترجمه کدهای خطای زرین‌پال به پیام‌های قابل فهم برای انسان.
             *
             * @param int|string $code کد خطای زرین‌پال.
             * @return string پیام خطای ترجمه شده.
             */
            private function get_error_message($code) {
                $errors = [
                    '-1'      => __('اطلاعات ارسال شده ناقص است.', 'wc-zarinpal-gateway'),
                    '-2'      => __('IP و یا مرچنت کد پذیرنده صحیح نیست.', 'wc-zarinpal-gateway'),
                    '-3'      => __('با توجه به محدودیت‌های شاپرک امکان پرداخت با رقم درخواست شده میسر نمی‌باشد.', 'wc-zarinpal-gateway'),
                    '-4'      => __('سطح تأیید پذیرنده پایین‌تر از سطح نقره‌ای است.', 'wc-zarinpal-gateway'),
                    '-11'     => __('درخواست مورد نظر یافت نشد.', 'wc-zarinpal-gateway'),
                    '-21'     => __('هیچ نوع عملیات مالی برای این تراکنش یافت نشد.', 'wc-zarinpal-gateway'),
                    '-22'     => __('تراکنش ناموفق می‌باشد.', 'wc-zarinpal-gateway'),
                    '-33'     => __('رقم تراکنش با رقم پرداخت شده مطابقت ندارد.', 'wc-zarinpal-gateway'),
                    '-34'     => __('سقف تقسیم تراکنش از لحاظ تعداد یا رقم عبور نموده است.', 'wc-zarinpal-gateway'),
                    '-40'     => __('اجازه دسترسی به متد مربوطه وجود ندارد.', 'wc-zarinpal-gateway'),
                    '-41'     => __('اطلاعات ارسال شده مربوط به AdditionalData نامعتبر می‌باشد.', 'wc-zarinpal-gateway'),
                    '-42'     => __('مدت زمان معتبر طول عمر شناسه پرداخت باید بین ۳۰ دقیقه تا ۴۵ روز باشد.', 'wc-zarinpal-gateway'),
                    '-54'     => __('درخواست مورد نظر آرشیو شده است.', 'wc-zarinpal-gateway'),
                    '100'     => __('عملیات با موفقیت انجام گردیده است.', 'wc-zarinpal-gateway'),
                    '101'     => __('عملیات پرداخت موفق بوده و قبلاً عملیات تأیید تراکنش انجام شده است.', 'wc-zarinpal-gateway'),
                    '-999'    => __('خطا در برقراری ارتباط با سرور زرین‌پال.', 'wc-zarinpal-gateway'),
                    'unknown' => __('خطای نامشخص رخ داده است.', 'wc-zarinpal-gateway'),
                ];

                return $errors[$code] ?? sprintf(__('خطای تعریف نشده با کد: %s', 'wc-zarinpal-gateway'), $code);
            }
        }
    }
    ```

4.  **آپلود آیکون (اختیاری اما توصیه می‌شود):** آیکون درگاه پرداخت زرین پال خود (مثلاً `zarinpal.png`) را در داخل پوشه `woocommerce-zarinpal-gateway`، کنار `woocommerce-zarinpal-gateway.php` قرار دهید. این آیکون در کنار نام درگاه پرداخت در صفحه تسویه‌حساب ظاهر می‌شود.
5.  **فعال‌سازی افزونه:** وارد پنل مدیریت وردپرس خود شوید، به بخش **"افزونه‌ها"** بروید و افزونه **"درگاه پرداخت زرین پال برای ووکامرس"** را **فعال کنید**.
6.  **پیکربندی درگاه:** به **ووکامرس > تنظیمات > پرداخت‌ها** بروید. اکنون باید "زرین پال" را در لیست مشاهده کنید. روی "مدیریت" (یا "راه‌اندازی") کلیک کنید تا **مرچنت کد** و سایر تنظیمات خود را پیکربندی نمایید.

## ملاحظات مهم

* **مرچنت کد:** برای استفاده از این درگاه **باید** یک مرچنت کد معتبر از زرین پال دریافت کنید.
* **واحد پول:** API زرین پال مبلغ را به **ریال** انتظار دارد. کد ارائه شده شامل منطق تبدیل تومان/IRT به ریال (`$amount *= 10`) است. اطمینان حاصل کنید که واحد پول ووکامرس شما به درستی تنظیم شده است و این تبدیل با انتظارات فعلی API زرین پال مطابقت دارد.
* **گواهی SSL:** برای امنیت، اطمینان حاصل کنید که وب‌سایت شما دارای گواهی SSL معتبر (HTTPS) است. درگاه‌های پرداخت نیاز به اتصالات امن دارند.
* **لاگ خطا:** فراخوانی‌های `error_log` برای اشکال‌زدایی مشکلات اتصال مفید هستند. در صورت بروز مشکل، لاگ خطاهای PHP سرور خود را بررسی کنید.
* **امنیت:** این یک پیاده‌سازی پایه است. برای محیط‌های تولید، همیشه بهترین روش‌های امنیتی اضافی را در نظر بگیرید، مانند لیست سفید IP (در صورت پشتیبانی توسط زرین پال) و ممیزی‌های امنیتی منظم.
* **به‌روزرسانی‌ها:** کد درگاه سفارشی نیاز به نگهداری دارد. اطمینان حاصل کنید که با هرگونه تغییر در API زرین پال یا استانداردهای درگاه پرداخت ووکامرس به‌روز می‌مانید.

## مشارکت (Contributing)

مشارکت شما خوشایند است! اگر پیشنهاد یا بهبودهایی برای این کد دارید، می‌توانید یک "Pull Request" ایجاد کنید یا "Issue" جدیدی را گزارش دهید.

## مجوز (License)

این پروژه تحت مجوز GPL-2.0-or-later منتشر شده است.
