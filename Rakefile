require 'tempfile'
require 'rake/clean'

task 'default' => '.screenshots'

#-----------------------------------------------------------------------------
# index
#-----------------------------------------------------------------------------

file 'fonts.dir' => ['.tamzen', '.powerline'] do
  sh 'mkfontdir'
  sh 'xset', '+fp', Dir.pwd
  sh 'xset', 'fp', 'rehash'
end
CLOBBER.include 'fonts.dir'

#-----------------------------------------------------------------------------
# fonts
#-----------------------------------------------------------------------------

class Font < Struct.new(:file, :props, :chars)
  def initialize file, contents
    head, *body, @tail = contents.split(/(?=\nSTARTCHAR|\nENDFONT)/)
    props = Hash[head.lines.map {|s| s.chomp.split(' ', 2) }.reject(&:empty?)]
    chars = Hash[body.map {|s| [Integer(s[/ENCODING (\d+)/, 1]), s] }]
    super file, props, chars

    # delete empty glyphs except space (32) and non-breaking space (160)
    chars.each do |code, char|
      if char =~ /BITMAP\n(?:0+\n)+ENDCHAR/ && code != 32 && code != 160
        chars.delete code
      end
    end
  end

  def to_s
    props['CHARS'] = chars.length
    [
      props.map {|*a| a.join(' ') }.join(?\n), ?\n,
      chars.values,
      @tail, ?\n
    ].join
  end
end

# Font files are looked up in the specified named git trees.
TAMZEN_BACKPORT_TREES = %w[ v1.6 v1.6-derived v1.9 ]

# For each font filename regexp, the specified names are substituted.
TAMZEN_BACKPORT_MOVES = {
  /8x16/ => '8x17',
  /7x13/ => '7x12',
}

# For each font filename regexp, the specified glyphs are backported.
TAMZEN_BACKPORT_SPECS = {
  # A B C D E F G H I J K L M N O P Q R S T U V W X Y Z   1 2 3 4 5
  # a b c d e f g h i j k l m n o p q r s t u v w x y z   6 7 8 9 0
  # { } [ ] ( ) < > $ * - + = / # _ % ^ @ \ & | ~ ? ' " ` ! , . ; :

  /(?#for-all-fonts)/ => %w[

      b           h       l m n   p q               y

  ],

  /10x20/ => %w[

                                O   Q
    a b c d e   g h   j   l m n o p q r     u   w   y

  ],

  /8x16/ => %w[

                                O   Q
    a b   d     g h       l m n   p q r     u   w   y

  ],

  /8x15/ => %w[

      b   d       h       l m n   p q r     u   w   y

  ],

  /7x14/ => %w[

      b           h     k l m n   p q   s           y

  ],

  /7x13/ => %w[

      b           h       l m n   p q           w   y

  ],
}

desc 'Build Tamzen fonts.'
file '.tamzen' => __FILE__ do
  require 'git'
  git = Git.open('.')

  # cache to speed up font lookups in loops below
  font_by_tree_and_file = Hash.new do |cache, key|
    tree, file = key
    if blob = git.gtree(tree).blobs[file]
      cache[key] = Font.new(file, blob.contents)
    end
  end

  # target the newest available Tamsyn release tag
  tags = git.tags.map(&:name).sort_by {|s| s.split(/[v.]/).map(&:to_i) }
  target_tag = tags.last

  # for each BDF font in the target release tag...
  git.gtree(target_tag).blobs.keys.grep(/\.bdf$/).each do |target_file|
    target_font = font_by_tree_and_file[[target_tag, target_file]]
    source_files = TAMZEN_BACKPORT_MOVES.map {|k,v| target_file.sub(k,v) }.
                   unshift(target_file).uniq

    # backport glyphs from previous font versions
    backport_glyphs = TAMZEN_BACKPORT_SPECS.
      select {|regexp, _| regexp =~ target_file }.
      map {|_, glyphs| glyphs }.flatten.uniq.sort

    backport_glyphs.all? do |glyph|
      codepoint = glyph.ord

      TAMZEN_BACKPORT_TREES.any? do |tree|
        source_files.any? do |file|
          source_font = font_by_tree_and_file[[tree, file]] and
          source_char = source_font.chars[codepoint] and
          target_font.chars[codepoint] = source_char # backport!
        end
      end or warn "#{target_file}: glyph #{glyph.inspect} (#{codepoint}) not found"
    end or warn "#{target_file}: not all glyphs were backported; see errors above"

    # save backported font under a different name
    rename = ['Tamsyn', 'Tamzen']
    File.write target_file.sub(*rename), target_font.to_s.gsub(*rename)
  end

  touch '.tamzen'
end
CLOBBER.include '.tamzen', '*.bdf'

desc 'Build Tamzen fonts for Powerline.'
file '.powerline' => ['.tamzen', 'bitmap-font-patcher'] do
  rename = [/Tamzen/, '\&ForPowerline']
  FileList['*.bdf'].exclude('*ForPowerline*').each do |src|
    dst = src.sub(*rename)
    IO.popen('python bitmap-font-patcher/fontpatcher.py', 'w+') do |patcher|
      patcher.write File.read(src).gsub(*rename).gsub('ISO8859', 'ISO10646')
      patcher.close_write
      File.write dst, Font.new(dst, patcher.read)
    end
  end
  touch '.powerline'
end
CLOBBER.include '.powerline'

file 'bitmap-font-patcher' do
  sh 'hg', 'clone', 'https://bitbucket.org/ZyX_I/bitmap-font-patcher'
end

#-----------------------------------------------------------------------------
# screenshots
#-----------------------------------------------------------------------------

desc 'Build font preview screenshots.'
file '.screenshots' => 'fonts.dir' do
  FileList['*.bdf'].ext('png').each do |png|
    Rake::Task[png].invoke
  end
  touch '.screenshots'
end
CLEAN.include '.screenshots', '*.png'

rule '.png' => ['.bdf', 'fonts.dir'] do |t|
  # translate the BDF font filename into its full X11 font name
  @bdf_to_x11 ||= Hash[File.readlines('fonts.dir').map(&:split)]

  # assemble sample text for rendering
  lines = [
    'ABCDEFGHIJKLMNOPQRSTUVWXYZ 12345',
    'abcdefghijklmnopqrstuvwxyz 67890',
    '{}[]()<>$*-+=/#_%^@\\&|~?\'"`!,.;:',
    #
    # visit the following URL for Unicode code points of powerline symbols
    # https://powerline.readthedocs.org/en/latest/fontpatching.html#glyph-table
    #
    "Illegal1i = oO0    \uE0A0 \uE0A1 \uE0A2 \uE0B0 \uE0B1 \uE0B2 \uE0B3"
  ]
  width = lines.first.length
  lines.unshift t.source.center(width)

  # store sample text in a file because it's the easiest way to render
  sample_text_file = Tempfile.open('screenshot')
  sample_text_file.write lines.join(?\n)
  sample_text_file.close

  # render sample text using the source font to the destination image
  sh 'xterm',
    '-fg', 'black',
    '-bg', 'white',
    '-T', t.source,
    '-font', @bdf_to_x11[t.source],
    '-geometry', "#{lines.first.length}x#{lines.length}",
    '-e', [
      'tput civis',                                 # hide the cursor
      "cat #{sample_text_file.path.inspect}",       # show sample text
      "import -window $WINDOWID #{t.name.inspect}", # take a screenshot
    ].join(' && ')
end
