require 'find'
require 'open3'
require 'yaml'
require 'ostruct'
require 'erb'
require 'rake/clean'

PARAMFILE = '_/param.yaml'

DEFAULTS  = {
  author: nil,
  description: nil,
  email: nil,
  url: nil,
  domain: nil,
  intro: '',
  googleanalytics: '',
  googleplus: '',
  brand: 'default',
  landslide: '/usr/bin/landslide',
  css: '_/default.css',
  js: '_/default.js',
  template: '_/index.erb',
}

class Hash
  def stringify
    inject({}) do |options, (key, value)|
      options[key.to_s] = value.to_s
      options
    end
  end

  def symbolize
    self.each_with_object({}) { |(k, v), h| h[k.to_sym] = v }
  end
end

def colorize(text, color_code)
  "\033[1m\033[38;5;#{color_code}m#{text}\033[0m"
end

def red(text);   colorize(text, 198); end
def green(text); colorize(text, 120); end
def blue(text);  colorize(text, 117); end

def run(*cmd)
  Open3.popen3(*cmd) do |_, out_p, err_p, thr_p|
    response, error = [out_p, err_p].map { |io| io.read.strip }
    status          = thr_p.value.exitstatus
    ok              = (status == 0)

    yield ok, response, error if block_given?

    status
  end
end

def erbify(infile, outfile, variables)
  template = File.read(infile)
  File.open(outfile, 'w') do |f|
    f.write ERB.new(template)
      .result(OpenStruct.new(variables).instance_eval { binding })
  end
end

#
# Folyoları 4'lü sütunlar olarak göstermek istiyoruz. Fakat CSS3'ün
# multi column spesifikasyonu tüm tarayıcılarda gerçeklenmemiş durumda.
# Örneğin 1..6 arasında isimlendirilmiş 6 adet folyomuz var.  İlgili CSS
# sınıfında column-count:4 # diyerek aşağıdaki gibi bir görüntü bekleriz.
#
# 	1 2 3 4
# 	5 6
#
# Pek çok tarayıcıda yerleşimi kontrol eden column-fill özelliği henüz
# gerçeklenmediğinden aşağıdaki gibi dengeli (balanced) yerleşimde tarama
# yapılıyor.
#
# 	1 3 5
# 	2 4 6
#
# Bu yerleşimde sadece 4'lü sütun görüntüsü kaybolmakla kalmıyor, soldan
# sağa sıralama da bozuluyor.  İmplementasyonda eksiklik olmasaydı
# öntanımlı değeri "balanced" olan "column-fill" niteliğini "auto" yaparak
# istediğimiz sonucu elde edecektik.  Bu olmadığından aşağıdaki dönüşümlerin
# yapıldığı geçici çözümü uyguluyoruz.
#
# 	[ 1 2 3 4 5 6 ] → [ 1 2 3 4 5 6 nil nil ] → [ 1 5 2 6 3 nil 4 nil ]
#
# P.S. nil değerlerini şablonda tespit ederek dolgu elemanına çeviriyoruz.
#
def pad(array, columns = 4)
  # dizi [1 2 3 4 5 6 ]
  result  = array
  # neme lazım
  return result if columns <= 0 || columns >= array.size
  # doldur →  [ 1 2 3 4 5 6 nil nil ]
  result += [nil] * (columns - array.size % columns)
  # transpozunu al → [ 1 5 2 6 3 nil 4 nil ]
  result.each_slice(columns).to_a.transpose.flatten
end

def itemize(items)
    pad(
      items.sort_by do |item|
        item[:label]
      end.map do |item|
        { label: item[:label], file: item[:destination] }
      end
    )
end

unless File.exist? PARAMFILE
  $stderr.puts red("Configuration file not found; creating a default one.")
  File.open(PARAMFILE, 'w') do |f|
    f.write(DEFAULTS.stringify.to_yaml)
  end
  $stderr.puts red("Please edit #{PARAMFILE}.")
  abort
end

Param = DEFAULTS.clone.merge YAML.load_file(PARAMFILE).symbolize

unless (unconfigured = Param.keys.select { |key| Param[key].nil? }).empty?
  $stderr.puts red(
    "Please supply values for the following parameters in #{PARAMFILE}:"
  ), "  #{unconfigured}"
  abort
end

unless File.exist? Param[:landslide]
  $stderr.puts red("#{Landslide} not found.")
  abort
end

# Css and Js files are not mandatory.
[Param[:css], Param[:js]].each do |f|
  unless File.exists? f
    touch f
    $stderr.puts red("Missing file #{f} created.")
  end
end

Items = []
FileList['[^_.]*'].select { |path| FileTest.directory?(path) }.each do |dir|
  Find.find(dir) do |path|
    basename = File.basename(path)
    if FileTest.directory?(path)
      Find.prune if %w(. _).any? { |prefix| basename[0] == prefix }
    elsif basename == 'index.md' && File.open(path, &:readline).start_with?('#')
      Items << {
        source: path,
        destination: path.ext('.html'),
        directory: File.dirname(path),
        label: File.dirname(path),
      }
    end
  end
end

# Create a file task for each destination.
Items.each do |item|
  file item[:destination] => [
    item[:source],
    Param[:css],
    Param[:js],
    *FileList["#{item[:directory]}/media/*"], # media files, i.e. images
    *FileList["#{item[:directory]}/code/*"],  # code files
    Param[:landslide]
  ] do
    $stderr.puts blue(item[:label])

    run(*%W[
      #{Param[:landslide]}
        --embed
        --linenos no
        --css #{Param[:css]}
        --js #{Param[:js]}
        --theme light
        --destination #{item[:destination]}
        #{item[:source]}
    ]) do |ok, response, error|
      if not ok
        rm_f item[:destination]
        $stderr.puts red("landslide error: %s" % error)
      end
    end
  end
end

destinations = Items.map { | item| item[:destination] }

desc 'Compile folios.'
task :compile => destinations
CLEAN.include destinations

file 'index.html'=> [PARAMFILE, Param[:template], *destinations] do
  $stderr.puts green('index')

  Param[:items] = itemize(Items)

  erbify(Param[:template], 'index.html', Param)
end

desc 'Index folios.'
task :index => 'index.html'
CLEAN.include 'index.html'

desc 'View index.'
task :view do
  sh 'xdg-open', 'index.html'
end

task :default => [:compile, :index]
