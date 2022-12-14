project(
  'libmali', 'c',
  version : '1.9.0',
  meson_version : '>=0.49.0',
  default_options : ['b_asneeded=false', 'b_lundef=false'])

mali_version = meson.project_version()

pkgconfig = import('pkgconfig')

if get_option('arch') != 'auto'
  arch = get_option('arch')
else
  arch = host_machine.cpu_family()
endif

gpu = get_option('gpu')
version = get_option('version')
subversion = get_option('subversion')
platform = get_option('platform')
opencl_icd = get_option('opencl-icd')
vendor_package = get_option('vendor-package')
wrapper_packages = get_option('wrapper-packages')

message('Building for ' + '|'.join([arch, gpu, version, subversion, platform]))

# Grab libraries with specified configs
cmd = run_command('grabber.sh', arch, gpu, version, subversion, platform)
libs = cmd.stdout().strip().split('\n')

# Use the first one as default library
default_lib = libs[0]
if default_lib == ''
  error('Failed to find matched library')
endif

message('Source libraries: @0@'.format(libs))

if wrapper_packages
  message('Provide wrapper packages')
else
  # The vendor package requires soname of wrapper packages.
  if vendor_package
    error('Cannot provide vendor package without wrapper packages')
  endif
endif

# Install wrapper libraries into vendor dir
if vendor_package
  message('Build vendor package')
  wrapper_libdir = get_option('libdir') / 'mali'
else
  wrapper_libdir = get_option('libdir')
endif

# Wrap library name : version
mali_wrappers = {'Mali' : '1'}
gbm_wrappers = {'gbm' : '1'}
egl_wrappers = {'EGL' : '1'}
glesv1_wrappers = {'GLESv1_CM' : '1'}
glesv2_wrappers = {'GLESv2' : '2'}
wayland_wrappers = {'wayland-egl' : '1'}
cl_wrappers = opencl_icd ? {'MaliOpenCL' : '1'} : {'OpenCL' : '1'}

# Source dir : dest dir
mali_headers = {
  'include/KHR' : 'KHR',
}
gbm_headers = {
  'include/GBM' : '',
}
egl_headers = {
  'include/EGL' : 'EGL',
}
glesv1_headers = {
  'include/GLES' : 'GLES',
}
glesv2_headers = {
  'include/GLES2' : 'GLES2',
  'include/GLES3' : 'GLES3',
}
cl_headers = {
  'include/CL' : 'CL',
}

# Provide newer GBM version with wrapper packages
gbm_version = wrapper_packages ? '20.1.5' : '10.4.0'

# Package name : required symbol, wrappers, headers, package version
map = {
  'Mali' : ['', mali_wrappers, mali_headers, mali_version],
  'gbm' : ['gbm_create_device', gbm_wrappers, gbm_headers, gbm_version],
  'egl' : ['eglCreateContext', egl_wrappers, egl_headers, '7.10'],
  'glesv1_cm' : ['eglCreateContext', glesv1_wrappers, glesv1_headers, '7.10'],
  'glesv2' : ['eglCreateContext', glesv2_wrappers, glesv2_headers, '7.10'],
  'wayland-egl' : ['wl_egl_window_create', wayland_wrappers, {}, '18.1.0'],
  'OpenCL' : ['clCreateContext', cl_wrappers, cl_headers, '1.2'],
}

# Create dummy source for building libraries
dummy_source = join_paths(meson.current_build_dir(), 'dummy.c')
run_command('touch', dummy_source)

# Create a dummy library for building wrappers
libmali = shared_library(
  'mali',
  dummy_source,
  install : true,
  version : mali_version)

pkgconfig.generate(
  libmali,
  description : 'Mali GPU User-Space Binary Driver')

# The gbm functions might be missing
gbm_check_funcs = [
  'gbm_bo_map',
  'gbm_bo_unmap',
  'gbm_bo_get_offset',
  'gbm_bo_get_plane_count',
  'gbm_device_get_format_modifier_plane_count',
  'gbm_bo_get_handle_for_plane',
  'gbm_bo_get_stride_for_plane',
  'gbm_bo_get_modifier',
  'gbm_bo_create_with_modifiers',
  'gbm_surface_create_with_modifiers',
  'gbm_bo_get_bpp',
  'gbm_format_get_name',
]

# Create libgbm wrapper for missing functions
libgbm = []
gbm_symbol = map['gbm'][0]
if run_command('grep', '-q', gbm_symbol, default_lib).returncode() == 0
  libgbm_version = gbm_wrappers['gbm']
  libgbm_cflags = [
    '-DLIBMALI_SO="libmali.so.' + mali_version.split('.')[0] + '"',
  ]

  libdrm_dep = dependency('libdrm', version : '>= 2.4.0')
  if not libdrm_dep.found()
    error('libdrm not found.')
  endif

  foreach symbol : gbm_check_funcs
    if run_command('grep', '-q', symbol, default_lib).returncode() == 0
      libgbm_cflags += '-DHAS_' + symbol
    endif
  endforeach

  libdl_dep = meson.get_compiler('c').find_library('dl', required : false)
  libgbm = shared_library(
    'gbm',
    'gbm_wrapper.c',
    c_args : libgbm_cflags,
    include_directories : include_directories('include/GBM'),
    dependencies : [libdl_dep, libdrm_dep],
    link_with : libmali,
    install : true,
    install_dir : wrapper_libdir,
    version : libgbm_version)
endif

foreach name, values : map
  symbol = values[0]
  wrappers = values[1]
  headers = values[2]
  pkg_version = values[3]
  mali_cflags = []
  wrapper_ldflags = []
  is_opencl_icd = opencl_icd and name == 'OpenCL'

  if run_command('grep', '-q', symbol, default_lib).returncode() != 0
    continue
  endif

  if name == 'egl' and platform != 'x11'
    mali_cflags += '-DMESA_EGL_NO_X11_HEADERS'
  endif

  foreach wrapper, version : wrappers
    wrapper_ldflags += '-l' + wrapper

    if wrapper != 'gbm'
      shared_library(
        wrapper,
        dummy_source,
        link_with : [libgbm, libmali],
        install : true,
        install_dir : wrapper_libdir,
        version : version)
    endif
  endforeach

  # Install ICD OpenCL vendor config
  if is_opencl_icd
    custom_target(
      'vendor icd',
      output : 'mali.icd',
      command : ['echo', 'libMaliOpenCL.so.1'],
      capture : true,
      install_dir : get_option('sysconfdir') / 'OpenCL' / 'vendors',
      install : true)
  endif

  # No {headers, pkgconfig} for {ICD OpenGL, vendor packages}
  if is_opencl_icd or vendor_package
    continue
  endif

  foreach src, dst : headers
    install_subdir(
      src,
      install_dir : get_option('includedir') / dst,
      install_mode : ['rw-r--r--', 'root'],
      strip_directory : true)
  endforeach

  pkgconfig.generate(
    libraries : ['-L${libdir} -lmali', wrapper_ldflags],
    extra_cflags : mali_cflags,
    version : pkg_version,
    name : name,
    description : 'Mali GPU User-Space Binary Driver Wrappers')
endforeach

# Install optional overlay
if get_option('with-overlay')
  if gpu == 'utgard-400' and subversion == 'r3p0'
    install_data('overlay/S10libmali_px3se', install_dir : get_option('sysconfdir') / 'init.d')
    install_data('overlay/px3seBase', install_dir : get_option('bindir'))
  endif

  if gpu == 'midgard-t76x' and subversion == 'all'
    install_data('overlay/S10libmali_rk3288', install_dir : get_option('sysconfdir') / 'init.d')
  endif
endif

if vendor_package
  # Install vendor ld config
  custom_target(
    'vendor ld config',
    output : '00-' + arch + '-mali.conf',
    command : ['echo', get_option('prefix') / wrapper_libdir],
    capture : true,
    install_dir : '/etc/ld.so.conf.d',
    install : true)
elif get_option('khr-header')
  # Install optional KHR header
  install_data(
    'include/KHR/mali_khrplatform.h',
    install_dir : get_option('includedir') / 'KHR',
    install_mode : ['rw-r--r--', 'root'],
    rename : 'khrplatform.h')
endif

# Install target libraries
install_data(libs, install_dir : get_option('libdir'))

# Fixup dummy library
meson.add_install_script('fixup_dummy.sh', get_option('libdir'), default_lib)

if not wrapper_packages
# Fixup wrappers
  meson.add_install_script('fixup_wrappers.sh', get_option('libdir'))
endif
