MeCab Win and MATLAB
====================

Trying out MeCab with MATLAB on Windows


###MeCab - Japanese Morphological Analyzer for text processing

When analyzing text, the first step usually involves getting the word counts. This is fairly easy with English as you can separate each word by whitespace. Japanese texts don't use whitespace characters, so this approach doesn't work. Hence you need a morphological analyzer to identify word separations. MeCab is one of the most commonly used open source morphological analyzers. You can download it from [Google Code](https://code.google.com/p/mecab/downloads/list).

###Installation

The easiest option is to download Windows executable [mecab-0.996.exe](https://code.google.com/p/mecab/downloads/detail?name=mecab-0.996.exe&can=2&q=). When you run the installer, make sure you select 'UTF-8' encoding. 

You can check if the installation was success as follows on MATLAB:

    % check to see if you can call MeCab. 
    % you should get "mecab of 0.996"
    system('mecab -v','-echo');
    clearvars ans
    
You should get `mecab of 0.996`

###Checking locale settings

On Windows MATLAB uses the locale settings from `Region and Language` control panel. If these are not set correctly, Japanese text may not be displayed correctly. Run `feature locale` at MATLAB prompt and make sure you get the following.

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


If not, go to `Control Panel` > `Region and Language` and change everything to Japanese: `Formats`, `Location`,  `Keyboards and Languages`, and finally `Administrative` > `Change system locale`. 

###Inserting whitespaces into Japanese text by word boundaries

Now you can use MeCab to insert whitespaces around words. MeCab is a command line tool. You pass the source text file and specify the output filename, and MeCab writes out the result to the specified file.

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

Now we have the processed Japanese text ready for tokenization and word count generation.

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

We want to randomize but place high word count words near the center. We can do this by using a point cloud with normal distribution that centers on mean. We can also control the spacing with standard deviation.

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
    hold off

MeCab WinとMATLAB
====================

Windows版のMeCabをMATLABから試してみました


###MeCab - テキスト処理のための日本語形態素解析ソフトウェア

テキスト解析を行う際に、最初の一歩は通常、語彙頻度の集計となります。英語の場合、スペースで単語を区切ることが出来るため、これはかなり容易です。日本語の場合、分かち書きを行わないため、この手法が使えません。よって、単語の分割に形態素解析ツールが必要となります。MeCabは、人気の高いオープンソースの形態素解析エンジンの一つです。[Google Code](https://code.google.com/p/mecab/downloads/list)からダウンロードできます。.

###インストール

一番簡単なのは、Windows版のexeファイル[mecab-0.996.exe](https://code.google.com/p/mecab/downloads/detail?name=mecab-0.996.exe&can=2&q=)をダウンロードすることです。.インストーラーを実行する際に、文字コードには'UTF-8'を選択して下さい。

インストールがうまくいったかどうかは、MATLABから以下のように確認できます:

    % check to see if you can call MeCab. 
    % you should get "mecab of 0.996"
    system('mecab -v','-echo');
    clearvars ans
    
`mecab of 0.996`と出れば問題ありません。

###ロケール設定を確認する

Windows上では、 MATLABは`地域と言語` コントロールパネルのロケール設定を参照します。これが正しく設定されていないと、日本語が正しく表示されないことがあります。MATLABのコマンドプロンプトで`feature locale`と入力し、以下と同じ設定になっていることを確認して下さい。

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


異なる場合は、`コントロールパネル` > `地域と言語` を開き、全てのタブで日本語を指定して下さい: `形式`, `場所`,  `キーボードと言語`, and finally `管理` > `システム ロケールの変更`

###日本語の文章を分かち書きに変換する

これでMeCabを使って単語を分かち書きする準備ができました。MeCabはコマンドラインツールです。変換したいファイルを引数にし、出力先ファイル名を指定すると、MeCabは指定されたファイルに変換結果を書き出します。

MeCabは、デフォルトで品詞情報なども出力しますが、今は分ち書きした文章のみで十分です。`-O wakati` オプションを使って、追加情報が出力されないようにします。

MeCabのホームページの最初の段落をUTF-8形式で`test.txt`にコピペしました。

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

コマンドウィンドウには、以下のように出力されました:

    Parsed file content:
    ﻿ MeCab ( 和布 蕪 ) と は 
    MeCab は 京都大 学 情報 学 研究 科 − 日本電信電話 株式会社 コミュニケーション 科学 基礎 研究所 共同 研究 ユニット  プロジェクト を通じて 開発 さ れ た オープン ソース 形態素 解析 エンジン です 。 言語 , 辞書 , コーパス に 依存 し ない 汎用 的 な 設計 を 基本 方針 と し て い ます 。 パラメータ の 推定 に Conditional Random Fields ( CRF ) を 用 い て おり , ChaSen が 採用 し て いる 隠れ マルコフ モデル に 比べ 性能 が 向上 し て い ます 。 また 、 平均 的 に ChaSen , Juman , KAKASI より 高速 に 動作 し ます 。 ちなみに 和布 蕪 ( め かぶ ) は , 作者 の 好物 です 。


###日本語のトークン分解

これで、トークン分解と語彙頻度集計に使える日本語テキストが用意できました。

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
    
###プロット表示用のデータ処理

日本語トークンと語彙頻度が用意できましたので、これをワードクラウドとして可視化すると面白いかもしれません。プロットに表示するために、追加のデータ処理を行います。

    % eliminate one-character words
    word_lengths=cellfun(@(x) length(x),words);
    key_words = words(word_lengths > 1);
    key_counts = counts(word_lengths > 1);
    % sort by word counts
    [~,IX] = sort(key_counts,'descend');
    % set font colors and sizes
    colors = colormap(hsv(length(key_words)));
    sizes = key_counts * 20;

###単語の表示位置を乱数で指定する

文字の表示位置を乱数によってしていますが、語彙頻度の高い単語はなるべく中心に集めたいと思います。0を平均とする正規分布のポイントクラウド（点群）を使えばいけそうです。点の間隔は標準偏差で調整できます。

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
    
###ワードクラウドをプロットする

これは、点の代わりに文字を使った散布図です。

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
    hold off

