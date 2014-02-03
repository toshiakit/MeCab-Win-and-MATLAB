MeCab Win and MATLAB
====================

Trying out MeCab with MATLAB on Windows


###MeCab - Japanese Morphological Analyzer for text processing

When analyzing text, the first step usually involves getting the word counts. This is fairly easy with English as you can separate each word by whitespace. Japanese text don't use whitespace characters, so this approach doesn't work. So you need a morphological analyzer to identify word separations. MeCab is one of the most commonly used open source morphological analyzer. You can download it from [Google Code](https://code.google.com/p/mecab/downloads/list).

###Installation

The easiest option is download Windows executable [mecab-0.996.exe](https://code.google.com/p/mecab/downloads/detail?name=mecab-0.996.exe&can=2&q=). When you run the installer, make sure you select 'UTF-8' encoding. 

Make sure installation was successful. You can test it on MATLAB as follows.

    % check to see if you can call MeCab. 
    % you should get "mecab of 0.996"
    system('mecab -v','-echo');
    clearvars ans
    
You should get `mecab of 0.996`

###Checking Locale settings

On Windows MATLAB uses the locale settings from Region and Langauge control panel. If these are not set correctly, Japanese text may not be displayed correctly. Run `feature locale` at MATLAB prompt and make sure you get the following.

               ctype: 'ja_JP.Shift_JIS'
             collate: 'ja_JP.Shift_JIS'
                time: 'ja_JP.Shift_JIS'
             numeric: 'en_US_POSIX.Shift_JIS'
            monetary: 'ja_JP.Shift_JIS'
            messages: 'ja_JP.Shift_JIS'
            encoding: 'Shift_JIS'
    terminalEncoding: 'Shift_JIS'
         jvmEncoding: 'MS932'
              status: 'MathWorks locale management system initialized.'
             warning: '


If not, go to Control Panel > Region and Langauge and change everything to Japanese: Format, Location, Keyboard and Language, and finally Advanced > System Locale. 

###Inserting whitespaces into Japanese text by word boundaries

Now you can use MeCab to insert whitespaces around words. MeCab is a command line tool. You pass the source text file and specify the output filename, and MeCab write out the result to the specified file.

MeCab also outputs other linguistical information by default, but we just need the processed text for now. So suppress other outputs by using `-O wakati` option.

I just copied and pasted the first paragraph of MeCab home page into `test.txt` in UTF-8 encoding.

    % get the current working directory
    dir = pwd;
    % specify input and output files
    infile = 'test.txt';outfile = 'out.txt';
    % convert to full file paths
    infile = fullfile(dir,infile);outfile = fullfile(dir,outfile);
    % string together the command to pass
    command = sprintf('mecab -O wakati %s > %s',infile,outfile);
    system(command,'-echo');
    if exist(outfile,'file') == 2
        fid = fopen(outfile,'r','n','UTF-8');
        if fid
            ja_text = fscanf(fid, '%c', inf);
            fclose(fid);
            disp('Parsed file content:')
            disp(ja_text)
        else
            fprintf('Unable to open %s\n', outfile);
            fclose(fid);
        end
    end
    clearvars ans command dir fid infile

The output to the command window was:

    Parsed file content:
    ﻿ MeCab ( 和布 蕪 ) と は 
    MeCab は 京都大 学 情報 学 研究 科 − 日本電信電話 株式会社 コミュニケーション 科学 基礎 研究所 共同 研究 ユニット  プロジェクト を通じて 開発 さ れ た オープン ソース 形態素 解析 エンジン です 。 言語 , 辞書 , コーパス に 依存 し ない 汎用 的 な 設計 を 基本 方針 と し て い ます 。 パラメータ の 推定 に Conditional Random Fields ( CRF ) を 用 い て おり , ChaSen が 採用 し て いる 隠れ マルコフ モデル に 比べ 性能 が 向上 し て い ます 。 また 、 平均 的 に ChaSen , Juman , KAKASI より 高速 に 動作 し ます 。 ちなみに 和布 蕪 ( め かぶ ) は , 作者 の 好物 です 。


###Tokenize the Japanese text

Now we have the processed Japanese text ready for tokenization and generate word counts.

    % specify the delimiters to separate text into tokens
    delims = {' ','@','$','/','#','.','-',':','&','*',...
        '+','=','[',']','?','!','(',')','{','}',',','''',...
        '"','>','_','<',';','%',char(10),char(13),char(8722),...
        char(12290),char(12289),char(65279)};
    % tokenize the content - returns a nested cell array
    tokens = textscan(ja_text,'%s','Delimiter',delims);
    % flatten the nested cell array
    tokens = tokens{:};
    % remove empty cells
    tokens = tokens(~cellfun('isempty',tokens));
    % remove duplicates
    [words,~,idx] = unique(tokens);
    % generate word counts 
    counts = accumarray(idx,1);
    clearvars delims idx ja_text outfile tokens
    
###Prepare data for plotting

Now that we have the tokenized text and word counts, perhaps it is interesting to visualize it as a word cloud. We need to further prepare our data for plotting.

    % eliminate one-character words
    word_lengths=cellfun(@(x) length(x),words);
    key_words = words(word_lengths > 1);
    key_counts = counts(word_lengths > 1);
    % sort by word counts
    [~,IX] = sort(key_counts,'descend');
    % set font colors and sizes
    colors = colormap(hsv(length(key_words)));
    sizes = key_counts * 20;

###Randomize the positions of words

We want to randomize but place high word count words near the center. We can do this by using a point cloud with normal distribution that centers around mean. We can also control the spacing with standard deviation.

    % initialize position matrix
    pos = zeros(length(key_words),1);
    % generate random points with normal distribution centered around 0
    pos(:,1) = randn(length(key_words),1);
    pos(:,2) = randn(length(key_words),1);
    [~,order] = sort(abs(pos(:,1).*pos(:,2)));
    pos = pos(order,:);
    % shift the center from 0 to 200 and spread out more
    mu = 200; % because the plot area will be 400 x 400
    std = 70; % this is just a guess
    pos = std .* pos + mu;
    
###Plot the word cloud

This is just a scatter plot where we place text instead of points.

    % plotting
    figure(1)
    xlim([0 400]); % x axis range 
    ylim([0 400]); % y axis range
    hold('all');
    title('MeCab Word Cloud');
    % loop through each word by word count
    for i=1:length(IX)
        x=pos(i,1);y=pos(i,2);
        str = key_words{IX(i)};
        color = colors(i,:);
        size = sizes(IX(i));
        h = text(x,y,str,'FontSize',size,'Color',color,...
            'HorizontalAlignment','center');
        % rotate text every other time
        if mod(i,2)==0
            set(h,'Rotation',270);
        end
    end


