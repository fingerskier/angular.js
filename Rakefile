require 'yaml'
include FileUtils

content = File.open('angularFiles.js', 'r') {|f| f.read }
files = eval(content.gsub(/angularFiles = /, '').gsub(/:/, '=>'));

BUILD_DIR = 'build'

task :default => [:compile, :test]


desc 'Init the build workspace'
task :init do
  FileUtils.mkdir(BUILD_DIR) unless File.directory?(BUILD_DIR)

  v = YAML::load( File.open( 'version.yaml' ) )
  match = v['version'].match(/^([^-]*)(-snapshot)?$/)

  NG_VERSION = Struct.new(:full, :major, :minor, :dot, :codename).
                      new(match[1] + (match[2] ? ('-' + %x(git rev-parse HEAD)[0..7]) : ''),
                          match[1].split('.')[0],
                          match[1].split('.')[1],
                          match[1].split('.')[2],
                          v['codename'])
end


desc 'Clean Generated Files'
task :clean do
  FileUtils.rm_r(BUILD_DIR, :force => true)
  FileUtils.mkdir(BUILD_DIR)
end


desc 'Compile Scenario'
task :compile_scenario => :init do

  deps = [
      'lib/jquery/jquery.js',
      'src/scenario/angular.prefix',
      files['angularSrc'],
      files['angularScenario'],
      'src/scenario/angular.suffix',
  ]

  concat = 'cat ' + deps.flatten.join(' ')

  File.open(path_to('angular-scenario.js'), 'w') do |f|
    f.write(%x{#{concat}}.gsub('"NG_VERSION_FULL"', NG_VERSION.full))
    f.write(gen_css('css/angular.css') + "\n")
    f.write(gen_css('css/angular-scenario.css'))
  end
end

desc 'Compile JSTD Scenario Adapter'
task :compile_jstd_scenario_adapter => :init do

  deps = [
      'src/jstd-scenario-adapter/angular.prefix',
      'src/jstd-scenario-adapter/Adapter.js',
      'src/jstd-scenario-adapter/angular.suffix',
  ]

  concat = 'cat ' + deps.flatten.join(' ')

  File.open(path_to('jstd-scenario-adapter.js'), 'w') do |f|
    f.write(%x{#{concat}}.gsub('"NG_VERSION_FULL"', NG_VERSION.full))
  end

  # TODO(vojta) use jstd configuration when implemented
  # (instead of including jstd-adapter-config.js)
  File.open(path_to('jstd-scenario-adapter-config.js'), 'w') do |f|
    f.write("/**\r\n" +
            " * Configuration for jstd scenario adapter \n */\n" +
            "var jstdScenarioAdapter = {\n  relativeUrlPrefix: '/build/docs/'\n};\n")
  end
end


desc 'Generate IE css js patch'
task :generate_ie_compat => :init do
  css = File.open('css/angular.css', 'r') {|f| f.read }

  # finds all css rules that contain backround images and extracts the rule name(s), content type of
  # the image and base64 encoded image data
  r = /\n([^\{\n]+)\s*\{[^\}]*background-image:\s*url\("data:([^;]+);base64,([^"]+)"\);[^\}]*\}/

  images = css.scan(r)

  # create a js file with multipart header containing the extracted images. the entire file *must*
  # be CRLF (\r\n) delimited
  File.open(path_to('angular-ie-compat.js'), 'w') do |f|
    f.write("/*\r\n" +
            "Content-Type: multipart/related; boundary=\"_\"\r\n" +
            "\r\n")

    images.each_index do |idx|
      f.write("--_\r\n" +
              "Content-Location:img#{idx}\r\n" +
              "Content-Transfer-Encoding:base64\r\n" +
              "\r\n" +
              images[idx][2] + "\r\n")
    end

    f.write("--_--\r\n" +
            "*/\r\n")

    # generate a css string containing *background-image rules for IE that point to the mime type
    # images in the header
    cssString = ''
    images.each_index do |idx|
      cssString += "#{images[idx][0]}{*background-image:url(\"mhtml:' + jsUri + '!img#{idx}\")}"
    end

    # generate a javascript closure that contains a function which will append the generated css
    # string as a stylesheet to the current html document
    jsString = "(function(){ \r\n" +
               "  var jsUri = document.location.href.replace(/\\/[^\\\/]+(#.*)?$/, '/') + \r\n" +
               "              document.getElementById('ng-ie-compat').src,\r\n" +
               "      css = '#{cssString}',\r\n" +
               "      s = document.createElement('style'); \r\n" +
               "\r\n" +
               "  s.setAttribute('type', 'text/css'); \r\n" +
               "\r\n" +
               "  if (s.styleSheet) { \r\n" +
               "    s.styleSheet.cssText = css; \r\n" +
               "  } else { \r\n" +
               "    s.appendChild(document.createTextNode(css)); \r\n" +
               "  } \r\n" +
               "  document.getElementsByTagName('head')[0].appendChild(s); \r\n" +
               "})();\r\n"

    f.write(jsString)
  end
end


desc 'Compile JavaScript'
task :compile => [:init, :compile_scenario, :compile_jstd_scenario_adapter, :generate_ie_compat] do

  deps = [
      'src/angular.prefix',
      files['angularSrc'],
      'src/angular.suffix',
  ]

  File.open(path_to('angular.js'), 'w') do |f|
    concat = 'cat ' + deps.flatten.join(' ')

    content = %x{#{concat}}.
              gsub('"NG_VERSION_FULL"', NG_VERSION.full).
              gsub('"NG_VERSION_MAJOR"', NG_VERSION.major).
              gsub('"NG_VERSION_MINOR"', NG_VERSION.minor).
              gsub('"NG_VERSION_DOT"', NG_VERSION.dot).
              gsub('"NG_VERSION_CODENAME"', NG_VERSION.codename).
              gsub(/^\s*['"]use strict['"];?\s*$/, ''). # remove all file-specific strict mode flags
              gsub(/'USE STRICT'/, "'use strict'")      # rename the placeholder in angular.prefix

    f.write(content)
    f.write(gen_css('css/angular.css', true))
  end

  %x(java -jar lib/closure-compiler/compiler.jar \
        --compilation_level SIMPLE_OPTIMIZATIONS \
        --language_in ECMASCRIPT5_STRICT \
        --js #{path_to('angular.js')} \
        --js_output_file #{path_to('angular.min.js')})

  FileUtils.cp_r 'i18n/locale', path_to('i18n')
end


desc 'Generate docs'
task :docs => [:init] do
  `node docs/src/gen-docs.js`
  File.open(path_to('docs/.htaccess'), File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('"NG_VERSION_FULL"', NG_VERSION.full)
  end
end


desc 'Create angular distribution'
task :package => [:clean, :compile, :docs] do
  tarball = "angular-#{NG_VERSION.full}.tgz"

  pkg_dir = path_to("pkg/angular-#{NG_VERSION.full}")
  FileUtils.rm_r(path_to('pkg'), :force => true)
  FileUtils.mkdir_p(pkg_dir)

  ['src/angular-mocks.js',
    path_to('angular.js'),
    path_to('angular.min.js'),
    path_to('angular-ie-compat.js'),
    path_to('angular-scenario.js'),
    path_to('jstd-scenario-adapter.js'),
    path_to('jstd-scenario-adapter-config.js'),
  ].each do |src|
    dest = src.gsub(/^[^\/]+\//, '').gsub(/((\.min)?\.js)$/, "-#{NG_VERSION.full}\\1")
    FileUtils.cp(src, pkg_dir + '/' + dest)
  end

  FileUtils.cp_r path_to('i18n'), "#{pkg_dir}/i18n-#{NG_VERSION.full}"
  FileUtils.cp_r path_to('docs'), "#{pkg_dir}/docs-#{NG_VERSION.full}"

  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/index.html", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('angular.min.js', "angular-#{NG_VERSION.full}.min.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end


  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/index-jq.html", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('angular.min.js', "angular-#{NG_VERSION.full}.min.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end


  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/index-nocache.html", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('angular.min.js', "angular-#{NG_VERSION.full}.min.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end


  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/index-jq-nocache.html", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('angular.min.js', "angular-#{NG_VERSION.full}.min.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end


  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/index-debug.html", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('../angular.js', "../angular-#{NG_VERSION.full}.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end


  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/index-jq-debug.html", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('../angular.js', "../angular-#{NG_VERSION.full}.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end

  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/docs-scenario.html", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('angular-scenario.js', "angular-scenario-#{NG_VERSION.full}.js")
  end

  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/appcache.manifest", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('angular.min.js', "angular-#{NG_VERSION.full}.min.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end

  File.open("#{pkg_dir}/docs-#{NG_VERSION.full}/appcache-offline.manifest", File::RDWR) do |f|
    text = f.read
    f.truncate 0
    f.rewind
    f.write text.sub('angular.min.js', "angular-#{NG_VERSION.full}.min.js").
                 sub('/build/docs/', "/#{NG_VERSION.full}/docs-#{NG_VERSION.full}/")
  end


  %x(tar -czf #{path_to(tarball)} -C #{path_to('pkg')} .)

  FileUtils.cp path_to(tarball), pkg_dir
  FileUtils.mv pkg_dir, path_to(['pkg', NG_VERSION.full])

  puts "Package created: #{path_to(tarball)}"
end


namespace :server do

  desc 'Run JsTestDriver Server'
  task :start do
    sh %x(java -jar lib/jstestdriver/JsTestDriver.jar --browser open --port 9876)
  end

  desc 'Run JavaScript tests against the server'
  task :test do
    sh %(java -jar lib/jstestdriver/JsTestDriver.jar --tests all)
  end

end


desc 'Run JavaScript tests'
task :test do
  sh %(java -jar lib/jstestdriver/JsTestDriver.jar --tests all --browser open --port 9876)
end


desc 'Lint'
task :lint do
  out = %x(lib/jsl/jsl -conf lib/jsl/jsl.default.conf)
  print out
end


desc 'push_angularjs'
task :push_angularjs => :compile do
  sh %(cat angularjs.ftp | ftp -N angularjs.netrc angularjs.org)
end



###################
# utility methods #
###################


##
# generates css snippet from a given files and optionally applies simple minification rules
#
def gen_css(cssFile, minify = false)
  css = ''
  File.open(cssFile, 'r') do |f|
    css = f.read
  end

  if minify
    css.gsub! /\n/, ''
    css.gsub! /\/\*.*?\*\//, ''
    css.gsub! /:\s+/, ':'
    css.gsub! /\s*\{\s*/, '{'
    css.gsub! /\s*\}\s*/, '}'
    css.gsub! /\s*\,\s*/, ','
    css.gsub! /\s*\;\s*/, ';'
  end

  #escape for js
  css.gsub! /\\/, "\\\\\\"
  css.gsub! /'/, "\\\\'"
  css.gsub! /\n/, "\\n"

  return %Q{angular.element(document).find('head').append('<style type="text/css">#{css}</style>');}
end


##
# returns path to the file in the build directory
#
def path_to(filename)
  return File.join(BUILD_DIR, *filename)
end
