<<<<<<< HEAD
require 'rake/clean'
require 'rake/testtask'
require 'fileutils'
require 'date'

# CI Reporter is only needed for the CI
begin
  require 'ci/reporter/rake/test_unit'
rescue LoadError
end

task :default => :test
task :spec => :test

CLEAN.include "**/*.rbc"

def source_version
  @source_version ||= begin
    load './lib/sinatra/version.rb'
    Sinatra::VERSION
  end
end

def prev_feature
  source_version.gsub(/^(\d\.)(\d+)\..*$/) { $1 + ($2.to_i - 1).to_s }
end

def prev_version
  return prev_feature + '.0' if source_version.end_with? '.0'
  source_version.gsub(/\d+$/) { |s| s.to_i - 1 }
end

# SPECS ===============================================================

task :test do
  ENV['LANG'] = 'C'
  ENV.delete 'LC_CTYPE'
end

Rake::TestTask.new(:test) do |t|
  t.test_files = FileList['test/*_test.rb']
  t.ruby_opts = ['-rubygems'] if defined? Gem
  t.ruby_opts << '-I.'
  t.warning = true
end

Rake::TestTask.new(:"test:core") do |t|
  core_tests = %w[base delegator encoding extensions filter
     helpers mapped_error middleware radius rdoc
     readme request response result route_added_hook
     routing server settings sinatra static templates]
  t.test_files = core_tests.map {|n| "test/#{n}_test.rb"}
  t.ruby_opts = ["-rubygems"] if defined? Gem
  t.ruby_opts << "-I."
  t.warning = true
end

# Rcov ================================================================

namespace :test do
  desc 'Measures test coverage'
  task :coverage do
    rm_f "coverage"
    sh "rcov -Ilib test/*_test.rb"
  end
end

# Website =============================================================

desc 'Generate RDoc under doc/api'
task 'doc'     => ['doc:api']
task('doc:api') { sh "yardoc -o doc/api" }
CLEAN.include 'doc/api'

# README ===============================================================

task :add_template, [:name] do |t, args|
  Dir.glob('README.*') do |file|
    code = File.read(file)
    if code =~ /^===.*#{args.name.capitalize}/
      puts "Already covered in #{file}"
    else
      template = code[/===[^\n]*Liquid.*index\.liquid<\/tt>[^\n]*/m]
      if !template
        puts "Liquid not found in #{file}"
      else
        puts "Adding section to #{file}"
        template = template.gsub(/Liquid/, args.name.capitalize).gsub(/liquid/, args.name.downcase)
        code.gsub! /^(\s*===.*CoffeeScript)/, "\n" << template << "\n\\1"
        File.open(file, "w") { |f| f << code }
      end
    end
  end
end

# Thanks in announcement ===============================================

team = ["Ryan Tomayko", "Blake Mizerany", "Simon Rozet", "Konstantin Haase"]
desc "list of contributors"
task :thanks, [:release,:backports] do |t, a|
  a.with_defaults :release => "#{prev_version}..HEAD",
    :backports => "#{prev_feature}.0..#{prev_feature}.x"
  included = `git log --format=format:"%aN\t%s" #{a.release}`.lines.map { |l| l.force_encoding('binary') }
  excluded = `git log --format=format:"%aN\t%s" #{a.backports}`.lines.map { |l| l.force_encoding('binary') }
  commits  = (included - excluded).group_by { |c| c[/^[^\t]+/] }
  authors  = commits.keys.sort_by { |n| - commits[n].size } - team
  puts authors[0..-2].join(', ') << " and " << authors.last,
    "(based on commits included in #{a.release}, but not in #{a.backports})"
end

desc "list of authors"
task :authors, [:commit_range, :format, :sep] do |t, a|
  a.with_defaults :format => "%s (%d)", :sep => ", ", :commit_range => '--all'
  authors = Hash.new(0)
  blake   = "Blake Mizerany"
  overall = 0
  mapping = {
    "blake.mizerany@gmail.com" => blake, "bmizerany" => blake,
    "a_user@mac.com" => blake, "ichverstehe" => "Harry Vangberg",
    "Wu Jiang (nouse)" => "Wu Jiang" }
  `git shortlog -s #{a.commit_range}`.lines.map do |line|
    line = line.force_encoding 'binary' if line.respond_to? :force_encoding
    num, name = line.split("\t", 2).map(&:strip)
    authors[mapping[name] || name] += num.to_i
    overall += num.to_i
  end
  puts "#{overall} commits by #{authors.count} authors:"
  puts authors.sort_by { |n,c| -c }.map { |e| a.format % e }.join(a.sep)
end

desc "generates TOC"
task :toc, [:readme] do |t, a|
  a.with_defaults :readme => 'README.md'

  def self.link(title)
    title.downcase.gsub(/(?!-)\W /, '-').gsub(' ', '-').gsub(/(?!-)\W/, '')
  end

  puts "* [Sinatra](#sinatra)"
  title = Regexp.new('(?<=\* )(.*)') # so Ruby 1.8 doesn't complain
  File.binread(a.readme).scan(/^##.*/) do |line|
    puts line.gsub(/#(?=#)/, '    ').gsub('#', '*').gsub(title) { "[#{$1}](##{link($1)})" }
  end
end

# PACKAGING ============================================================

if defined?(Gem)
  # Load the gemspec using the same limitations as github
  def spec
    require 'rubygems' unless defined? Gem::Specification
    @spec ||= eval(File.read('sinatra.gemspec'))
  end

  def package(ext='')
    "pkg/sinatra-#{spec.version}" + ext
  end

  desc 'Build packages'
  task :package => %w[.gem .tar.gz].map {|e| package(e)}

  desc 'Build and install as local gem'
  task :install => package('.gem') do
    sh "gem install #{package('.gem')}"
  end

  directory 'pkg/'
  CLOBBER.include('pkg')

  file package('.gem') => %w[pkg/ sinatra.gemspec] + spec.files do |f|
    sh "gem build sinatra.gemspec"
    mv File.basename(f.name), f.name
  end

  file package('.tar.gz') => %w[pkg/] + spec.files do |f|
    sh <<-SH
      git archive \
        --prefix=sinatra-#{source_version}/ \
        --format=tar \
        HEAD | gzip > #{f.name}
    SH
  end

  task 'release' => ['test', package('.gem')] do
    if File.binread("CHANGES") =~ /= \d\.\d\.\d . not yet released$/i
      fail 'please update changes first' unless %x{git symbolic-ref HEAD} == "refs/heads/prerelease\n"
    end

    sh <<-SH
      gem install #{package('.gem')} --local &&
      gem push #{package('.gem')}  &&
      git commit --allow-empty -a -m '#{source_version} release'  &&
      git tag -s v#{source_version} -m '#{source_version} release'  &&
      git tag -s #{source_version} -m '#{source_version} release'  &&
      git push && (git push sinatra || true) &&
      git push --tags && (git push sinatra --tags || true)
    SH
  end
end
=======
# encoding: utf-8
require 'rake/clean'
require 'rdoc'
require 'rdoc/encoding'
require 'rdoc/markup/to_html'
require 'rdoc/options'
require 'uri'
require 'nokogiri'
require 'kramdown'


def cleanup(html)
  html_doc = Nokogiri::HTML(html)
  html_doc.xpath("//h1").first.remove # Removes Sinatra Heading
  html_doc.xpath("//ul").first.remove # Removes the ToC in Markdown
  toc_header = html_doc.xpath("//h2").first
  toc_header.remove if toc_header.inner_html == "Table of Contents"
  html_doc.to_html
end


def readme(pattern = "%s", &block)
  return readme(pattern).each(&block) if block_given?
  %w[en de es fr hu ja zh ru ko pt-br pt-pt].map do |lang|
    pattern % "README#{lang == "en" ? "" : ".#{lang}"}"
  end
end

def contrib(pattern = "%s", &block)
  return contrib(pattern).each(&block) if block_given?

  %w[
    sinatra-config-file
    sinatra-multi-route
    sinatra-content-for
    sinatra-namespace
    sinatra-cookies
    sinatra-reloader
    sinatra-decompile
    sinatra-respond-with
    sinatra-extension
    sinatra-streaming
    sinatra-json
    sinatra-link-header
  ].map do |extension|
    pattern % extension
  end
end

# generates Table of Contents
def with_toc(src)
  toc = "<div class='toc'>\n"
  last_level = 1
  src = src.gsub(/<h(\d)>(.*)<\/h\d>/) do |line|
    level, heading = $1.to_i, $2.strip
    aname = URI.escape(heading)
    toc << "\t"*last_level << "<ol class='level-#{level-1}'>\n" if level > last_level
    toc << "\t"*level << "</ol>"*(last_level - level) << "\n" if level < last_level
    toc << "\t"*level << "<li><a href='##{aname}'>#{heading}</a></li>\n"
    last_level = level
    "<a name='#{aname}'></a>\n" << line
  end
  toc << "\t" << "</ol>"*(last_level-1) << "\n</div>\n"
  toc + src
end

task :default => ['_sinatra', '_contrib', :build]

desc "Build outdated static files and API docs"
task :build => [:pull, 'build:static']

desc "Build outdated static files"
task 'build:static' => readme("_includes/%s.html") + contrib("_includes/%s.html")

desc "Build anything that's outdated and stage changes for next commit"
task :regen => [:build] do
  sh 'git add api _includes'
  puts "\nPrebuilt files regenerated and staged in your index. Commit with:"
  puts "  git commit -m 'Regen prebuilt files'"
end

desc 'Pull in the latest from the sinatra and sinatra-contrib repos'
task :pull => ['pull:sinatra', 'pull:contrib']

directory "_sinatra" do
  puts 'Cloning sinatra repo'
  sh "git clone git://github.com/sinatra/sinatra.git _sinatra" 
end

desc 'Pull in the latest from the sinatra repo'
task 'pull:sinatra' => "_sinatra" do
    puts 'Pulling sinatra.git'
    sh "cd _sinatra && git pull &>/dev/null"
end

directory "_contrib" do
  puts 'Cloning sinatra-contrib repo'
  sh "git clone git://github.com/sinatra/sinatra-contrib.git _contrib" 
end

desc 'Pull in the latest from the sinatra-contrib repo'
task 'pull:contrib' => "_contrib" do
    puts 'Pulling sinatra-contrib.git'
    sh "cd _contrib && git pull &>/dev/null"
end

readme("_sinatra/%s.md") { |fn| file fn => '_sinatra' }
file 'AUTHORS' => '_sinatra'

readme do |fn|
  file "_includes/#{fn}.html" => ["_sinatra/#{fn}.md", "Rakefile"] do |f|
    markdown_string = File.read("_sinatra/#{fn}.md").encode('UTF-16le', :invalid => :replace, :replace => "").encode("UTF-8")
    markdown_string.gsub!(/```(\s?(\w+\n))?/) do |match|
      match =~ /```\s?\n/ ? "~~~~~\n" : match.sub(/```\s?/, "~~~~")
    end
    markdown = Kramdown::Document.new(markdown_string, :fenced_code_blocks => true, :coderay_line_numbers => nil, :auto_ids => false)
    html = cleanup(markdown.to_html)
    File.open(f.name, 'w') { |io| io.write with_toc(html) }
  end
end


desc 'Build contrib docs'
task 'build:contrib_docs' => 'pull:contrib' do
  puts 'Building sinatra-contrib docs'
  sh "cd _contrib && rake doc &>/dev/null"
end


contrib("_contrib/doc/%s.rdoc") { |fn| file fn => '_contrib' }

contrib do |fn|
  file "_includes/#{fn}.html" => ["build:contrib_docs", "_contrib/doc/#{fn}.rdoc", "Rakefile"] do |f|
    html =
      RDoc::Markup::ToHtml.new(RDoc::Options.new)
      .convert(File.read("_contrib/doc/#{fn}.rdoc"))
    File.open(f.name, 'wb') { |io| io.write html }
  end
end

desc 'Rebuild site under _site with Jekyll'
task :jekyll do
  rm_rf '_site'
  sh 'jekyll build'
end

desc 'Start the Jekyll server on http://localhost:4000/'
task :server do
  rm_rf '_site'
  puts 'jekyll serve --watch'
  exec 'jekyll serve --watch'
end

CLEAN.include '_site', "_includes/*.html"
CLOBBER.include "_contrib", "_sinatra"
>>>>>>> upstream/master
