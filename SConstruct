# rippled SConstruct
#
'''

    Target          Builds
    ----------------------------------------------------------------------------

    <none>          Same as 'install'
    install         Default target and copies it to build/rippled (default)

    all             All available variants
    debug           All available debug variants
    release         All available release variants

    clang           All clang variants
    clang.debug     clang debug variant
    clang.release   clang release variant

    gcc             All gcc variants
    gcc.debug       gcc debug variant
    gcc.release     gcc release variant

    msvc            All msvc variants
    msvc.debug      MSVC debug variant
    msvc.release    MSVC release variant

    vcxproj         Generate Visual Studio 2013 project file

If the clang toolchain is detected, then the default target will use it, else
the gcc toolchain will be used. On Windows environments, the MSVC toolchain is
also detected.

'''
#
'''

TODO

- Fix git-describe support


'''
#-------------------------------------------------------------------------------

import collections
import os
import subprocess
import sys
import textwrap
import SCons.Action

sys.path.append(os.path.join('src', 'beast', 'site_scons'))

import Beast

OSX_OPENSSL_ROOT = '/usr/local/Cellar/openssl/'

#-------------------------------------------------------------------------------

def osx_openssl_path():
    most_recent = sorted(os.listdir(OSX_OPENSSL_ROOT))[-1]
    return os.path.join(OSX_OPENSSL_ROOT, most_recent)

def gcc_is_clang():
    try:
        return 'clang' in subprocess.check_output(['g++', '--version'])
    except:
        return False

# Display command line exemplars
def print_coms(target, source, env):
    print ('Target: ' + Beast.yellow(str(target[0])))
    # TODO Add 'PROTOCCOM' to this list and make it work
    Beast.print_coms(['CXXCOM', 'CCCOM', 'LINKCOM'], env)

def detect_toolchains(env):
    toolchains = []
    if env.Detect('clang'):
        toolchains.append('clang')
    if env.Detect('g++') and not gcc_is_clang():
        toolchains.append('gcc')
    if env.Detect('cl'):
        toolchains.append('msvc')
    return toolchains

def files(base):
    for parent, _, files in os.walk(base):
        for path in files:
            path = os.path.join(parent, path)
            yield os.path.normpath(path)

def category(ext):
    if ext in ['.c', '.cc', '.cpp']:
        return 'compiled'
    return 'none'

def unity_category(f):
    base, fullname = os.path.split(f)
    name, ext = os.path.splitext(fullname)
    if os.path.splitext(name)[1] == '.unity':
        return category(ext)
    return 'none'

def categorize(groups, func, sources):
    for f in sources:
        groups.setdefault(func(f), []).append(f)

#-------------------------------------------------------------------------------

# Set construction variables for the base environment
def config_base(env):
    env.Replace(
        CCCOMSTR='Compiling ' + Beast.blue('$SOURCES'),
        CXXCOMSTR='Compiling ' + Beast.blue('$SOURCES'),
        LINKCOMSTR='Linking ' + Beast.blue('$TARGET'),
        )
    #git = Beast.Git(env) #  TODO(TOM)
    if False: #git.exists:
        env.Append(CPPDEFINES={'GIT_COMMIT_ID' : '"%s"' % git.commit_id})
    if Beast.system.linux:
        env.ParseConfig('pkg-config --static --cflags --libs openssl')
        env.ParseConfig('pkg-config --static --cflags --libs protobuf')
    elif Beast.system.windows:
        env.Append(CPPPATH=[os.path.join('src', 'protobuf', 'src')])
    elif Beast.system.osx:
        openssl = osx_openssl_path()
        env.Prepend(CPPPATH='%s/include' % openssl)
        env.Prepend(LIBPATH=['%s/lib' % openssl])

# Set toolchain and variant specific construction variables
def config_env(toolchain, variant, env):
    try:
        BOOST_ROOT = os.path.normpath(os.environ['BOOST_ROOT'])
        env.Append(CPPPATH=[BOOST_ROOT])
        env.Append(LIBPATH=[os.path.join(BOOST_ROOT, 'stage', 'lib')])
    except KeyError:
        pass

    try:
        OPENSSL_ROOT = os.path.normpath(os.environ['OPENSSL_ROOT'])
        env.Append(CPPPATH=[OPENSSL_ROOT])
    except KeyError:
        pass

    if variant == 'debug':
        env.Append(CPPDEFINES=['DEBUG', '_DEBUG'])

    elif variant == 'release':
        env.Append(CPPDEFINES=['NDEBUG'])

    if toolchain in Split('clang gcc'):
        env.Append(CCFLAGS=[
            '-Wall',
            '-Wno-sign-compare',
            '-Wno-char-subscripts',
            '-Wno-format',
            ])

        env.Append(CXXFLAGS=[
            '-frtti',
            '-std=c++11',
            '-Wno-invalid-offsetof'])

        if Beast.system.osx:
            env.Append(CCFLAGS=[
                '-Wno-deprecated',
                '-Wno-deprecated-declarations',
                '-Wno-unused-variable',
                '-Wno-unused-function',
                ])
            env.Append(CPPDEFINES={'BEAST_COMPILE_OBJECTIVE_CPP': 1})
        else:
            env.Append(CCFLAGS=['-Wno-unused-but-set-variable'])

        env.Append(LIBS=[
            'boost_date_time',
            'boost_filesystem',
            'boost_program_options',
            'boost_regex',
            'boost_system',
            'boost_thread',
            'dl',
            ])
        if Beast.system.osx:
            env.Append(LIBS=['crypto', 'protobuf', 'ssl'])
        else:
            env.Append(LIBS=['rt'])

        env.Append(LINKFLAGS=['-rdynamic'])

        if variant == 'debug':
            env.Append(CCFLAGS=['-g'])
        elif variant == 'release':
            env.Append(CCFLAGS=['-O3', '-fno-strict-aliasing'])

        if toolchain == 'clang':
            env.Replace(CC='clang', CXX='clang++', LINK='g++')
            # C and C++
            # Add '-Wshorten-64-to-32'
            env.Append(CCFLAGS=[])
            # C++ only
            env.Append(CXXFLAGS=['-Wno-mismatched-tags'])

        elif toolchain == 'gcc':
            env.Replace(CC='gcc', CXX='g++')
            env.Append(CCFLAGS=['-Wno-unused-local-typedefs'])

        if Beast.system.osx:
            env.Append(FRAMEWORKS=['AppKit', 'Foundation'])

    elif toolchain == 'msvc':
        env.Append (CPPPATH=[
            os.path.join('src', 'protobuf', 'src'),
            os.path.join('src', 'protobuf', 'vsprojects'),
            ])
        env.Append(CCFLAGS=[
            '/bigobj',              # Increase object file max size
            '/EHa',                 # ExceptionHandling all
            '/fp:precise',          # Floating point behavior
            '/Gd',                  # __cdecl calling convention
            '/Gm-',                 # Minimal rebuild: disabled
            '/GR',                  # Enable RTTI
            '/Gy-',                 # Function level linking: disabled
            '/MP',                  # Multiprocessor compilation
            '/openmp-',             # pragma omp: disabled
            '/Zc:forScope',         # Language extension: for scope
            '/Zi',                  # Generate complete debug info
            '/errorReport:none',    # No error reporting to Internet
            '/nologo',              # Suppress login banner
            '/Fd${TARGET}.pdb',     # Path: Program Database (.pdb)
            '/W3',                  # Warning level 3
            '/WX-',                 # Disable warnings as errors
            '/wd"4018"',            # Disable warning C4018
            '/wd"4244"',            # Disable warning C4244
            '/wd"4267"',            # Disable warning 4267
            ])
        env.Append(CPPDEFINES={
            '_WIN32_WINNT' : '0x6000',
            })
        env.Append(CPPDEFINES=[
            '_SCL_SECURE_NO_WARNINGS',
            '_CRT_SECURE_NO_WARNINGS',
            'WIN32_CONSOLE',
            ])
        env.Append(LIBS=[
            'ssleay32MT.lib',
            'libeay32MT.lib',
            'Shlwapi.lib',
            'kernel32.lib',
            'user32.lib',
            'gdi32.lib',
            'winspool.lib',
            'comdlg32.lib',
            'advapi32.lib',
            'shell32.lib',
            'ole32.lib',
            'oleaut32.lib',
            'uuid.lib',
            'odbc32.lib',
            'odbccp32.lib',
            ])
        env.Append(LIBPATH='D:\lib\OpenSSL-Win64\lib\VC\static')
        env.Append(LINKFLAGS=[
            '/DEBUG',
            '/DYNAMICBASE',
            '/ERRORREPORT:NONE',
            #'/INCREMENTAL',
            '/MACHINE:X64',
            '/MANIFEST',
            #'''/MANIFESTUAC:"level='asInvoker' uiAccess='false'"''',
            #'/NOLOGO',
            '/NXCOMPAT',
            '/SUBSYSTEM:CONSOLE',
            '/TLBID:1',
            ])

        if variant == 'debug':
            env.Append(CCFLAGS=[
                '/GS',              # Buffers security check: enable
                '/MTd',             # Language: Multi-threaded Debug CRT
                '/Od',              # Optimization: Disabled
                '/RTC1',            # Run-time error checks:
                ])
            env.Append(CPPDEFINES=[
                '_CRTDBG_MAP_ALLOC'
                ])
        else:
            env.Append(CCFLAGS=[
                '/MT',              # Language: Multi-threaded CRT
                '/Ox',              # Optimization: Full
                ])

    else:
        raise SCons.UserError('Unknown toolchain == "%s"' % toolchain)

#-------------------------------------------------------------------------------

# Configure the base construction environment
root_dir = Dir('#').srcnode().get_abspath() # Path to this SConstruct file
build_dir = os.path.join('build')
base = Environment(
    toolpath=[os.path.join ('src', 'beast', 'site_scons', 'site_tools')],
    tools=['default', 'Protoc', 'VSProject'],
    ENV=os.environ,
    TARGET_ARCH='x86_64')
config_base(base)
base.Append(CPPPATH=[
    'src',
    os.path.join('src', 'leveldb'),
    os.path.join('src', 'leveldb', 'port'),
    os.path.join('src', 'leveldb', 'include'),
    os.path.join(build_dir, 'proto')])

# Configure the toolchains, variants, default toolchain, and default target
toolchains = detect_toolchains(base)
variants = ['debug', 'release']
default_variant = 'debug'
default_target = None

# Collect sources from recursive directory iteration
groups = collections.defaultdict(list)
categorize(groups, unity_category,
    list(files('src/ripple')) +
    list(files('src/ripple_app')) +
    list(files('src/ripple_basics')) +
    list(files('src/ripple_core')) +
    list(files('src/ripple_data')) +
    list(files('src/ripple_hyperleveldb')) +
    list(files('src/ripple_leveldb')) +
    list(files('src/ripple_net')) +
    list(files('src/ripple_overlay')) +
    list(files('src/ripple_rpc')) +
    list(files('src/ripple_websocket')))
groups['protoc'].append (
    os.path.join('src', 'ripple', 'proto', 'ripple.proto'))
for source in groups['protoc']:
    outputs = base.Protoc([],
        source,
        PROTOCPROTOPATH=[os.path.dirname(source)],
        PROTOCOUTDIR=os.path.join(build_dir, 'proto'),
        PROTOCPYTHONOUTDIR=None)
    groups['none'].extend(outputs)
if Beast.system.windows:
    groups['compiled'].append (os.path.join('src', 'protobuf.unity.cpp'))
if Beast.system.osx:
    mm = os.path.join('src', 'ripple', 'beast', 'ripple_beastobjc.unity.mm')
    srcs.append(mm)

# Declare the targets
aliases = collections.defaultdict(list)
msvc_configs = []
for toolchain in toolchains:
    for variant in variants:
        # Configure this variant's construction environment
        env = base.Clone()
        config_env(toolchain, variant, env)
        variant_name = '%s.%s' % (toolchain, variant)
        variant_dir = os.path.join(build_dir, variant_name)
        variant_dirs = {
            os.path.join(variant_dir, 'src') :
                'src',
            os.path.join(variant_dir, 'proto') :
                os.path.join (build_dir, 'proto'),
            }
        for dest, source in variant_dirs.iteritems():
            env.VariantDir(dest, source, duplicate=0)
        objects = [env.Object(x) for x in Beast.variantFiles(
            groups['compiled'], variant_dirs)]
        target = env.Program(
            target = os.path.join(variant_dir, 'rippled'),
            source = objects
            )
        #print_action = env.Command(variant_name, [], Action(print_coms, ''))
        #env.Depends(objects, print_action)
        if (env.get('CC', None) == base.get('CC', None) and
                variant == default_variant):
            default_target = target
            install_target = env.Install (build_dir, source = default_target)
            env.Alias ('install', install_target)
            env.Default (install_target)
            aliases['all'].append(install_target)
        if toolchain == 'msvc':
            config = env.VSProjectConfig(variant, 'x64', target, env)
            msvc_configs.append(config)
        aliases['all'].append(target)
        aliases[variant].append(target)
        aliases[toolchain].append(target)
        env.Alias(variant_name, target)
for key, value in aliases.iteritems():
    env.Alias(key, value)

vcxproj = env.VSProject(
    os.path.join('Builds', 'VisualStudio2013', 'RippleD2'),
    source = groups['compiled'],
    VSPROJECT_ROOT_DIRS = ['src'],
    VSPROJECT_EXCLUDED_SOURCE = groups['none'],
    VSPROJECT_CONFIGS = msvc_configs)
env.Alias('vcxproj', vcxproj)
