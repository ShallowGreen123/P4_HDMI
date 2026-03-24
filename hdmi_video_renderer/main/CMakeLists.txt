idf_component_register(
    SRCS "main.c" "app_stream_adapter.c" "app_extractor.c"
    INCLUDE_DIRS "."
    REQUIRES
        esp_driver_jpeg
        esp_driver_ppa
        esp_extractor
        esp_mm
        esp_timer
        t5_p4_board)
