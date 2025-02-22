project(
	'swaylock',
	'c',
	version: '1.7',
	license: 'MIT',
	meson_version: '>=0.59.0',
	default_options: [
		'c_std=c11',
		'warning_level=2',
		'werror=true',
	],
)

add_project_arguments(
	[
		'-Wno-unused-parameter',
		'-Wno-unused-result',
		'-Wundef',
		'-Wvla',
	],
	language: 'c',
)

cc = meson.get_compiler('c')

sysconfdir = get_option('sysconfdir')
prefix = get_option('prefix')
is_freebsd = host_machine.system().startswith('freebsd')

add_project_arguments(
	'-DSYSCONFDIR="/@0@"'.format(join_paths(prefix, sysconfdir)),
	language : 'c',
)

if is_freebsd
	add_project_arguments('-D_C11_SOURCE', language: 'c')
endif

wayland_client = dependency('wayland-client', version: '>=1.20.0')
wayland_protos = dependency('wayland-protocols', version: '>=1.25', fallback: 'wayland-protocols')
wayland_scanner = dependency('wayland-scanner', version: '>=1.15.0', native: true)
xkbcommon = dependency('xkbcommon')
cairo = dependency('cairo')
gdk_pixbuf = dependency('gdk-pixbuf-2.0', required: get_option('gdk-pixbuf'))
bash_comp = dependency('bash-completion', required: false)
fish_comp = dependency('fish', required: false)
libpam = cc.find_library('pam', required: get_option('pam'))
crypt = cc.find_library('crypt', required: not libpam.found())
math = cc.find_library('m')

git = find_program('git', required: false)
scdoc = find_program('scdoc', required: get_option('man-pages'))
wayland_scanner_prog = find_program(wayland_scanner.get_variable('wayland_scanner'), native: true)

version = '"@0@"'.format(meson.project_version())
if git.found()
	git_commit_hash = run_command([git, 'describe', '--always', '--tags'], check: false)
	git_branch = run_command([git, 'rev-parse', '--abbrev-ref', 'HEAD'], check: false)
	if git_commit_hash.returncode() == 0 and git_branch.returncode() == 0
		version = '"@0@ (" __DATE__ ", branch \'@1@\')"'.format(git_commit_hash.stdout().strip(), git_branch.stdout().strip())
	endif
endif
add_project_arguments('-DSWAYLOCK_VERSION=@0@'.format(version), language: 'c')

wl_protocol_dir = wayland_protos.get_variable('pkgdatadir')

wayland_scanner_code = generator(
	wayland_scanner_prog,
	output: '@BASENAME@-protocol.c',
	arguments: ['private-code', '@INPUT@', '@OUTPUT@'],
)

wayland_scanner_client = generator(
	wayland_scanner_prog,
	output: '@BASENAME@-client-protocol.h',
	arguments: ['client-header', '@INPUT@', '@OUTPUT@'],
)

client_protos_src = []
client_protos_headers = []

client_protocols = [
	wl_protocol_dir / 'stable/xdg-shell/xdg-shell.xml',
	wl_protocol_dir / 'staging/ext-session-lock/ext-session-lock-v1.xml',
	'wlr-layer-shell-unstable-v1.xml',
	'wlr-input-inhibitor-unstable-v1.xml',
]

foreach xml : client_protocols
	client_protos_src += wayland_scanner_code.process(xml)
	client_protos_headers += wayland_scanner_client.process(xml)
endforeach

lib_client_protos = static_library(
	'client_protos',
	client_protos_src + client_protos_headers,
	dependencies: [wayland_client]
) # for the include directory

client_protos = declare_dependency(
	link_with: lib_client_protos,
	sources: client_protos_headers,
)

conf_data = configuration_data()
conf_data.set10('HAVE_GDK_PIXBUF', gdk_pixbuf.found())

subdir('include')

dependencies = [
	cairo,
	client_protos,
	gdk_pixbuf,
	math,
	xkbcommon,
	wayland_client,
]

sources = [
	'background-image.c',
	'cairo.c',
	'comm.c',
	'log.c',
	'loop.c',
	'main.c',
	'password.c',
	'password-buffer.c',
	'pool-buffer.c',
	'render.c',
	'seat.c',
	'unicode.c',
]

if libpam.found()
	sources += ['pam.c']
	dependencies += [libpam]
else
	warning('The swaylock binary must be setuid when compiled without libpam')
	warning('You must do this manually post-install: chmod a+s /path/to/swaylock')
	sources += ['shadow.c']
	dependencies += [crypt]
endif

swaylock_inc = include_directories('include')

executable('swaylock',
	sources,
	include_directories: [swaylock_inc],
	dependencies: dependencies,
	install: true
)

install_data(
	'pam/swaylock',
	install_dir: sysconfdir + '/pam.d/'
)

if scdoc.found()
	mandir = get_option('mandir')
	man_files = [
		'swaylock.1.scd',
	]
	foreach filename : man_files
		topic = filename.split('.')[-3].split('/')[-1]
		section = filename.split('.')[-2]
		output = '@0@.@1@'.format(topic, section)

		custom_target(
			output,
			input: filename,
			output: output,
			command: scdoc,
			feed: true,
			capture: true,
			install: true,
			install_dir: '@0@/man@1@'.format(mandir, section)
		)
	endforeach
endif

datadir = get_option('datadir')

if get_option('zsh-completions')
	zsh_files = files(
		'completions/zsh/_swaylock',
	)
	zsh_install_dir = datadir + '/zsh/site-functions'

	install_data(zsh_files, install_dir: zsh_install_dir)
endif

if get_option('bash-completions')
	bash_files = files(
		'completions/bash/swaylock',
	)
	if bash_comp.found()
		bash_install_dir = bash_comp.get_variable('completionsdir')
	else
		bash_install_dir = datadir + '/bash-completion/completions'
	endif

	install_data(bash_files, install_dir: bash_install_dir)
endif

if get_option('fish-completions')
	fish_files = files(
		'completions/fish/swaylock.fish',
	)
	if fish_comp.found()
		fish_install_dir = fish_comp.get_variable('completionsdir')
	else
		fish_install_dir = datadir + '/fish/vendor_completions.d'
	endif

	install_data(fish_files, install_dir: fish_install_dir)
endif
