#
# Copyright (c) 2022 Nordic Semiconductor
#
# SPDX-License-Identifier: LicenseRef-Nordic-5-Clause
#

add_library(fp_crypto STATIC)

if(CONFIG_BT_FAST_PAIR_CRYPTO)
  target_sources(fp_crypto PRIVATE fp_crypto_common.c)
endif()
if(CONFIG_BT_FAST_PAIR_CRYPTO_MBEDTLS)
  target_sources(fp_crypto PRIVATE fp_crypto_mbedtls.c)
endif()
if(CONFIG_BT_FAST_PAIR_CRYPTO_TINYCRYPT)
  target_sources(fp_crypto PRIVATE fp_crypto_tinycrypt.c)
endif()

target_include_directories(fp_crypto PUBLIC include)
target_include_directories(fp_crypto PUBLIC ../include/common)

target_link_libraries(fp_crypto PRIVATE zephyr_interface)
