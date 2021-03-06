# Copyright © 2017 Intel Corporation

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

anv_entrypoints = custom_target(
  'anv_entrypoints.[ch]',
  input : ['anv_entrypoints_gen.py', vk_api_xml, vk_android_native_buffer_xml],
  output : ['anv_entrypoints.h', 'anv_entrypoints.c'],
  command : [prog_python2, '@INPUT0@', '--xml', '@INPUT1@',
             '--xml', '@INPUT2@', '--outdir', meson.current_build_dir()],
  depend_files : files('anv_extensions.py'),
)

anv_extensions_c = custom_target(
  'anv_extensions.c',
  input : ['anv_extensions.py', vk_api_xml, vk_android_native_buffer_xml],
  output : 'anv_extensions.c',
  command : [prog_python2, '@INPUT0@', '--xml', '@INPUT1@',
             '--xml', '@INPUT2@', '--out', '@OUTPUT@'],
)

intel_icd = custom_target(
  'intel_icd',
  input : 'anv_icd.py',
  output : 'intel_icd.@0@.json'.format(target_machine.cpu()),
  command : [prog_python2, '@INPUT@',
             '--lib-path', join_paths(get_option('prefix'), get_option('libdir')),
             '--out', '@OUTPUT@'],
  depend_files : files('anv_extensions.py'),
  build_by_default : true,
  install_dir : with_vulkan_icd_dir,
  install : true,
)

# TODO: workaround for anv_entrypoints combining the .h and .c files in it's
# output. See issue #2346
block_entrypoints = custom_target(
  'block_entrypoints',
  command : [prog_touch, '@OUTPUT@'],
  output : 'null',
  depends : anv_entrypoints,
)

libanv_gen_libs = []
anv_gen_files = files(
  'genX_blorp_exec.c',
  'genX_cmd_buffer.c',
  'genX_gpu_memcpy.c',
  'genX_pipeline.c',
  'genX_query.c',
  'genX_state.c',
)
foreach g : [['70', ['gen7_cmd_buffer.c']], ['75', ['gen7_cmd_buffer.c']],
             ['80', ['gen8_cmd_buffer.c']], ['90', ['gen8_cmd_buffer.c']],
             ['100', ['gen8_cmd_buffer.c']]]
  _gen = g[0]
  _files = g[1]
  _lib = static_library(
    'libanv_gen@0@'.format(_gen),
    [anv_gen_files, _files, block_entrypoints, nir_opcodes_h],
    include_directories : [inc_common, inc_compiler, inc_drm_uapi, inc_intel,
                           inc_vulkan_util, inc_vulkan_wsi],
    c_args : [c_vis_args, no_override_init_args, '-msse2',
              '-DGEN_VERSIONx10=@0@'.format(_gen)],
    dependencies : [dep_libdrm, dep_valgrind],
  )
  libanv_gen_libs += _lib
endforeach

libanv_files = files(
  'anv_allocator.c',
  'anv_batch_chain.c',
  'anv_blorp.c',
  'anv_cmd_buffer.c',
  'anv_debug_report.c',
  'anv_descriptor_set.c',
  'anv_device.c',
  'anv_dump.c',
  'anv_formats.c',
  'anv_genX.h',
  'anv_image.c',
  'anv_intel.c',
  'anv_nir.h',
  'anv_nir_apply_pipeline_layout.c',
  'anv_nir_lower_input_attachments.c',
  'anv_nir_lower_multiview.c',
  'anv_nir_lower_push_constants.c',
  'anv_pass.c',
  'anv_pipeline.c',
  'anv_pipeline_cache.c',
  'anv_private.h',
  'anv_queue.c',
  'anv_util.c',
  'anv_wsi.c',
  'vk_format_info.h',
)

anv_deps = []
anv_flags = []

if with_platform_x11
  anv_deps += dep_xcb_dri3
  anv_flags += [
    '-DVK_USE_PLATFORM_XCB_KHR',
    '-DVK_USE_PLATFORM_XLIB_KHR',
  ]
  libanv_files += files('anv_wsi_x11.c')
endif

if with_platform_wayland
  anv_deps += dep_wayland_client
  anv_flags += '-DVK_USE_PLATFORM_WAYLAND_KHR'
  libanv_files += files('anv_wsi_wayland.c')
endif

libanv_common = static_library(
  'anv_common',
  [libanv_files, anv_entrypoints, anv_extensions_c, nir_opcodes_h],
  include_directories : [inc_common, inc_intel, inc_compiler, inc_drm_uapi,
                         inc_vulkan_util, inc_vulkan_wsi],
  c_args : [c_vis_args, no_override_init_args, '-msse2', anv_flags],
  dependencies : [dep_valgrind],
)

libvulkan_intel = shared_library(
  'vulkan_intel',
  [files('anv_gem.c'), block_entrypoints],
  include_directories : [inc_common, inc_intel, inc_compiler, inc_drm_uapi,
                         inc_vulkan_util, inc_vulkan_wsi],
  link_whole : [libanv_common, libanv_gen_libs],
  link_with : [libintel_compiler, libintel_common, libisl, libblorp,
               libvulkan_util, libvulkan_wsi, libnir, libmesa_util],
  dependencies : [dep_libdrm, dep_thread, dep_dl, dep_m, anv_deps, dep_valgrind],
  c_args : [c_vis_args, no_override_init_args, '-msse2', anv_flags],
  link_args : ['-Wl,--build-id=sha1', ld_args_bsymbolic, ld_args_gc_sections],
  install : true,
)

if with_tests
  libvulkan_intel_test = static_library(
    'vulkan_intel_test',
    [files('anv_gem_stubs.c'), block_entrypoints],
    include_directories : [inc_common, inc_intel, inc_compiler, inc_drm_uapi,
                           inc_vulkan_util, inc_vulkan_wsi],
    link_whole : libanv_common,
    link_with : [libanv_gen_libs, libintel_compiler, libintel_common,
                 libisl, libblorp, libvulkan_util, libvulkan_wsi,
                 libnir, libmesa_util],
    dependencies : [dep_libdrm, dep_thread, dep_dl, dep_m, anv_deps,
                    dep_valgrind],
    c_args : [c_vis_args, no_override_init_args, '-msse2', anv_flags],
  )

  foreach t : ['block_pool_no_free', 'state_pool_no_free',
               'state_pool_free_list_only', 'state_pool']
    _exe = executable(
      t,
      ['tests/@0@.c'.format(t), dummy_cpp, block_entrypoints],
      link_with : libvulkan_intel_test,
      dependencies : [dep_libdrm, dep_thread, dep_m, dep_valgrind],
      include_directories : [inc_common, inc_intel, inc_compiler,
                             inc_vulkan_util, inc_vulkan_wsi],
    )
    test('anv_@0@'.format(t), _exe)
  endforeach
endif
