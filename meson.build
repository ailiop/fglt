project('fglt', 'cpp',
        version : '1.0.1',
        license : 'GPL-3.0-or-later',
        meson_version : '>=0.50.0',
        default_options: ['buildtype=release',
                          'default_library=both'])

# ========== compiler options

cc = meson.get_compiler('cpp')

is_icpc         = cc.get_id() == 'intel' or cc.get_id() == 'intel-cl'
enable_opencilk = cc.has_header('cilk/cilk.h') and cc.has_argument('-fopencilk')
enable_cilkplus = not enable_opencilk \
                  and cc.has_header('cilk/cilk.h') \
                  and (is_icpc or cc.has_argument('-fcilkplus'))

if is_icpc
  add_global_arguments(['-wd3947,3946,10006,3950'], language : 'cpp')
endif

if enable_opencilk
  add_global_arguments(['-fopencilk', '-DHAVE_CILK_CILK_H'], language : 'cpp')
  add_global_link_arguments(['-fopencilk'], language : 'cpp')
elif enable_cilkplus
  add_global_arguments(['-fcilkplus', '-DHAVE_CILK_CILK_H'], language : 'cpp')
  add_global_link_arguments(['-lcilkrts'], language : 'cpp')
endif

# ========== OpenCilk tools

args_cs           = []
args_link_cs      = []
str_cs            = ''
args_bench        = []
args_link_bench   = []
str_bench         = ''
args_san          = []
# FIXME Separate `_lib`,`_bin` flags due to temporary issue in OpenCilk 1.0-rc1
args_link_san_lib = []
args_link_san_bin = []
str_san           = ''

if get_option('cilktool') == 'cilkscale' # ----- Cilkscale
  if not enable_opencilk
    warning('Cilkscale tool specified but not using OpenCilk compiler;'
            + ' skipping instrumentation...')
  else
    args_cs         = ['-fcilktool=cilkscale', '-DCILKSCALE']
    args_link_cs    = ['-fcilktool=cilkscale']
    str_cs          = '-cilkscale'
    args_bench      = ['-fcilktool=cilkscale-benchmark', '-DCILKSCALE']
    args_link_bench = ['-fcilktool=cilkscale-benchmark']
    str_bench       = str_cs + '-bench'
  endif
endif

if get_option('cilktool') == 'cilksan' # ----- Cilksan
  if not enable_opencilk
    warning('Cilksan tool specifiied but not using OpenCilk compiler'
            + ' skipping instrumentation...')
  else
    if not get_option('debug')
      warning('Cilksan instrumentation specified but not building in debug mode!')
    endif
    args_san          = ['-fsanitize=cilk']
    args_link_san_lib = ['-fsanitize=cilk']
    # FIXME There does not seem to be a way to add static-only arguments to
    # build targets defined via `library()`.
    # See also: https://github.com/mesonbuild/meson/issues/3304
    if not get_option('replicate_bug_libasan') \
       and get_option('default_library') != 'static'
      args_link_san_lib += ['-shared-libasan']
    endif
    args_link_san_bin = ['-fsanitize=cilk']
    str_san           = '-cilksan'
  endif
endif

# ========== build targets

fglthpp = configure_file(copy: true,
                         input: 'lib/fglt.hpp',
                         output: 'fglt.hpp')

install_headers('lib/fglt.hpp')

fglt_lib = library('fglt' + str_cs + str_san, 'lib/fglt.cpp',
                   cpp_args : args_cs + args_san,
                   link_args : args_link_cs + args_link_san_lib,
                   install : true, install_dir : 'lib',
                   soversion : '0')

e = executable('fglt' + str_cs + str_san, 'src/fglt_mtx.cpp',
               link_with : fglt_lib,
               cpp_args : args_cs + args_san,
               link_args : args_link_cs + args_link_san_bin,
               install : true,
               install_rpath : get_option('prefix') / 'lib')

if args_bench != []
  fglt_lib_bench = library('fglt' + str_bench, 'lib/fglt.cpp',
                           cpp_args : args_bench,
                           link_args : args_link_bench,
                           install : true, install_dir : 'lib',
                           soversion : '0')
  e_bench = executable('fglt' + str_bench, 'src/fglt_mtx.cpp',
                       link_with : fglt_lib_bench,
                       cpp_args : args_bench,
                       link_args : args_link_bench,
                       install : true,
                       install_rpath : get_option('prefix') / 'lib')
endif

# ========== tests

t1 = find_program( 'testdata/test_s6.sh')
t2 = find_program( 'testdata/test_s12.sh')
test('Validation on s6 synthetic graph', t1, args : e, is_parallel : false)
test('Validation on s12 synthetic graph', t2, args : e, is_parallel : false)

# ========== benchmark

if get_option('enable_benchmark')

  # FIXME The test-matrix file should not be downloaded anew for each build
  # FIXME The test-matrix file should not be copied into each build directory
  dat_bench_tgz = configure_file(output : 'coPapersDBLP.tar.gz',
                                 command : ['wget', 'https://suitesparse-collection-website.herokuapp.com/MM/DIMACS10/coPapersDBLP.tar.gz'])
  dat_bench = configure_file(output : 'coPapersDBLP.mtx',
                             input : dat_bench_tgz,
                             command : ['tar', '-xvzf', dat_bench_tgz, '--strip', '1'],
                             install : true, install_dir : 'testdata')

  benchmark('suitesparse/coPapersDBLP', e, args : dat_bench)

endif
