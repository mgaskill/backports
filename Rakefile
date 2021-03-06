begin
  require "bundler/gem_tasks"
rescue LoadError
  # bundler not installed
end

require 'rake/testtask'
Rake::TestTask.new(:test) do |test|
  test.libs << 'lib' << 'test'
  test.pattern = 'test/**/*_test.rb'
  test.verbose = false
end

class SpecRunner
  STATS = [:files, :examples, :expectations, :failures, :errors]
  attr_reader :stats, :not_found

  def initialize
    @counts = [0] * 5
    @not_found = []
  end

  def run(cmd, path)
    result = `#{cmd}`
    match = result.match(/(\d+) files?, (\d+) examples?, (\d+) expectations?, (\d+) failures?, (\d+) errors?/)
    if match.nil?
      puts "*** mspec returned with unexpected results:"
      puts result
      puts "Command was:", cmd
    end
    _, ex, p, f, e = data = match.captures.map{|x| x.to_i}
    not_found << path if ex == 0
    STATS.each_with_index do |_, i|
      @counts[i] += data[i]
    end
    if f + e > 0
      puts cmd
      puts result
    else
      print "."
      STDOUT.flush
    end
  end

  def stats
    h = {}
    STATS.zip(@counts).each{|k, v| h[k]=v}
    h
  end

  def report
    puts "*** Overall:", stats.map{|a| a.join(' ')}.join(', ')
    puts "No spec found for #{@not_found.join(', ')}" unless @not_found.empty?
  end

  def success?
    stats[:failures] == 0 && stats[:errors] == 0
  end
end

desc "Run specs, where path can be '*/*' (default), 'class/*' or 'class/method'."
task :spec, :path, :action do |t, args|
  args.with_defaults(:path => '*/*', :action => 'ci')
  specs = SpecRunner.new
  mspec_cmds(args[:path], 'frozen_old_spec', args[:action]) do |cmd, path|
    specs.run(cmd, path)
  end
  mspec_cmds(args[:path], 'spec', args[:action]) do |cmd, path|
    specs.run(cmd, path)
  end
  specs.report
  fail unless specs.success?
end

task :all_spec do # Necessary because of argument passing bug in 1.8.7
  Rake::Task[:spec].invoke
end

task :spec_tag, :path do |t, args|
  Rake::Task[:spec].invoke(args[:path], 'tag -G fails')
end

task :default => [:test, :all_spec]

DEPENDENCIES = Hash.new([]).merge!(
  '1.8.7/argf/chars'     => 'backports/1.8.7/string/each_char',
  '1.8.7/argf/each_char' => 'backports/1.8.7/string/each_char',
  '1.8.7/array/cycle'    => 'backports/1.8.7/stop_iteration',
  '1.8.7/enumerable/entries' => ['backports/1.8.7/enumerable/each_with_index', 'backports/1.8.7/enumerable/to_a'],
  '1.8.7/enumerator/rewind' => 'backports/1.8.7/enumerator/next',
  '1.8.7/hash/reject' => 'backports/1.8.7/integer/even',
  '1.9.1/hash/rassoc'    => 'backports/1.9.1/hash/key',
  '1.9.1/proc/lambda'    => 'backports/1.9.1/proc/curry',
  '1.9.2/complex/to_r'   => 'complex',
  '1.9.2/array/select'    => 'backports/1.8.7/array/select',
  '1.9.2/hash/select'    => 'backports/1.8.7/hash/select',
  '1.9.2/enumerable/each_entry' => 'backports/1.8.7/enumerable/each_with_index',
  '2.0.0/hash/to_h'      => 'backports/1.9.1/hash/default_proc',
  '2.2.0/float/next_float' => 'backports/2.2.0/float/prev_float',
  '2.2.0/float/prev_float' => 'backports/2.2.0/float/next_float'
)
{
  :each_with_index => %w[enumerable/detect enumerable/find enumerable/find_all enumerable/select enumerable/to_a],
  :first => %w[enumerable/cycle io/bytes io/chars io/each_byte io/each_char io/lines io/each_line]
}.each do |req, libs|
  libs.each{|l| DEPENDENCIES["1.8.7/#{l}"] = "backports/1.8.7/enumerable/#{req}" }
end

# These **old** specs cause actual errors while loading in 1.8:
OLD_IGNORE_IN_18 = %w[
  1.9.1/symbol/length
  1.9.1/symbol/size
  1.9.3/string/byteslice
  1.8.7/proc/yield
  1.9.1/proc/case_compare
]
# These **new** specs cause actual errors while loading in 1.8:
IGNORE_IN_18 = %w[
  2.1.0/enumerable/to_h
  2.1.0/array/to_h
  2.1.0/module/include
]
def mspec_cmds(pattern, spec_folder, action='ci')
  pattern = "lib/backports/*.*.*/#{pattern}.rb"
  Dir.glob(pattern) do |lib_path|
    _match, version, path = lib_path.match(/backports\/(\d\.\d\.\d)\/(.*)\.rb/).to_a
    next if path =~ /stdlib/
    next if version <= RUBY_VERSION
    version_path = "#{version}/#{path}"
    if RUBY_VERSION <= '2.0.0'
      next if OLD_IGNORE_IN_18.include? version_path if RUBY_VERSION < '1.9'
      next if IGNORE_IN_18.include? version_path
      next if spec_folder != 'frozen_old_spec' && version <= '2.0.0'  # Don't run new specs for pre 2.0 features & ruby
    end
    deps = [*DEPENDENCIES[version_path]].map{|p| "-r #{p}"}.join(' ')
    klass, method = path.split('/')
    path = [klass.gsub('_', ''), method].join('/') # don't ask me why RubySpec uses matchdata instead of match_data
    yield %W[mspec #{action}
              -I lib
              -r ./set_version/#{version}
              #{deps}
              -r backports/#{version_path}
              #{spec_folder}/rubyspec/core/#{path}_spec.rb
            ].join(' '), path
  end
end
