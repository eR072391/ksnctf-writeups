## Crawling Chaos

![](img/crawling-chaos1.png)

問題ページにある"unya.html"にアクセスしてみる。  

![](img/crawling-chaos2.png)

まずは、適当な文字を入力して挙動を調べる。  
だが、何も表示されない。  

次に、リクエストとレスポンスを見てみる。  

![](img/crawling-chaos3.png)

POSTもGETでも入力した文字を送信していなかった。  
ソースコードを見てみる。  

![](img/crawling-chaos4.png)

なんだか、可愛らしい顔文字が見える。  
(ᒧᆞωᆞ)=(/ᆞωᆞ/),(ᒧᆞωᆞ).ᒧうー=-!!(/ᆞωᆞ/).にゃー

scriptタグに囲まれているところから、何かしらのプログラムコードだと予想。  

調べてみると、[Brainf**k](https://ja.wikibooks.org/wiki/Brainfuck)というプログラム言語だそう。  
```> < + - . , [ ]```を使用した、8つの命令しかない。  

コードの部分をコピーしてjsファイルとして保存。  
nodeコマンドで実行してみる。  

![](img/crawling-chaos5.png)

エラーが発生したが、行おうとしたコードが出力されている。  

```$(function(){$("form").submit(function(){var t=$('input[type="text"]').val();var p=Array(70,152,195,284,475,612,791,896,810,850,737,1332,1469,1120,1470,832,1785,2196,1520,1480,1449);var f=false;if(p.length==t.length){f=true;for(var i=0;i<p.length;i++)if(t.charCodeAt(i)*(i+1)!=p[i])f=false;if(f)alert("(」・ω・)」うー!(/・ω・)/にゃー!");}if(!f)alert("No");return false;});});```

見やすく、整形した結果。  

```
$(function() {
	$("form").submit(function() {
		var t = $('input[type="text"]').val();
		var p = Array(70, 152, 195, 284, 475, 612, 791, 896, 810, 850, 737, 1332,
			1469, 1120, 1470, 832, 1785, 2196, 1520, 1480, 1449);
		var f = false;
		if (p.length == t.length) {
			f = true;
			for (var i = 0; i < p.length; i++)
				if (t.charCodeAt(i) * (i + 1) != p[i]) f = false;
			if (f) alert("(」・ω・)」うー!(/・ω・)/にゃー!");
		}
		if (!f) alert("No");
		return false;
	});
});
```

入力された文字（t）とpのそれぞれの要素が等しいか調べている。  
等しい場合は```(」・ω・)」うー!(/・ω・)/にゃー!```、等しくない場合に```No```とalertが発生する。  

入力した文字は```charCodeAt()```によって、UTF-13文字コードに変換され、i+1を掛けて何やら重み付けをしている。  
pの各要素の、数値にする前の文字を逆算して調べる。  
具体的にはindex+1を掛けてるなら、掛けた数で割って戻す。  


```
var p = [70, 152, 195, 284, 475, 612, 791, 896, 810, 850, 737, 1332,
         1469, 1120, 1470, 832, 1785, 2196, 1520, 1480, 1449];

var result = "";
for (var i = 0; i < p.length; i++) {
    result += String.fromCharCode(p[i] / (i + 1));
}
console.log(result);
```

flagを得られた。  






