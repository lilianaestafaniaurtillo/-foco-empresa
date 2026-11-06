jQuery(function () {
    jQuery(document).on('gform_post_render', function () {
        jQuery('.sidebar-form input[type=file]').on('change', function () {
            const fileName = jQuery(this).val().split('\\').pop();
            jQuery('.gfield_description', jQuery(this).parent().parent()).html(fileName);
        });
    });
});

jQuery(function () {
    jQuery(document).on("click", "[data-scroll-to]", function () {
        var $this = jQuery(this),
            $target = jQuery($this.attr("data-scroll-to"));

        if ($target.length > 0) {
            jQuery("html, body").animate({
                scrollTop: $target.offset().top - 120
            }, 500);
        }
    });
});
