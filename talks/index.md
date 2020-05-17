---
layout: default
title: Talks
---
<a href="https://amitlan.github.io/writing">Writing</a> | <b>Talks</b> | <a href="https://amitlan.github.io/photolog">Pictures</a> | <a href="https://amitlan.github.io/bookmarks">Links</a>
<hr>
<table cellspacing="15" class="posts">
  {% for post in site.tags["talks"] %}
  <tr>
    <td><a href="{{ post.external_url }}"><b>{{ post.title }}</b></a></td><td>{{ post.description}}</td>
  </tr>
  {% endfor %}
</table>
<table cellspacing="15">
  <tr>	  
    <td>
		<p>
			<a href="https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/slides/pgconf-eu-2019.pdf">
				<b>Partitioning in Postgres: How Far We've Come</b>
			</a>
		</p>
	    	<p>(Slides are slightly different from those used in Bali for the same talk)</p>
	</td>
  
    <td><p>
       PGConf.ASIA 2019, Milan (Italy)
    </p></td>
  </tr>
  <tr>
    <td>
		<p>
			<a href="https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/slides/pgconf-asia-2019.pdf">
				<b>Partitioning in Postgres: How Far We've Come</b>
			</a>
		</p>
	</td>
  
	  		<td><p>
			PGConf.ASIA 2019, Bali (Indonesia)
		</p></td>

  </tr>
  <tr>
    <td>
		<p>
			<a href="https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/slides/pg-11-partition-shines-pgconf-asia-2018.pdf">
				<b>Partitioning Shines in PostgreSQL 11</b>
			</a>
		</p>
	</td>
	
		<td><p>
			PGConf.ASIA 2018, Tokyo
		</p></td>

</tr>
  <tr>
    <td>
		<p>
			<a href="https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/slides/pg-built-in-sharding-pgconf-asia-2017.pdf">
				<b>PostgreSQL Built-in Sharding</b>
			</a>
		</p>
	</td>
	
		<td><p>
			PGConf.ASIA 2017, Tokyo
		</p></td>

</tr>
  <tr>
    <td>
		<p>
			<a href="https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/slides/pg-decl-part-arrived-pgconf-asia-2017.pdf">
				<b>Declarative Partitioning Has Arrived!</b>
			</a>
		</p>
	</td>
	
		<td><p>
			PGConf.ASIA 2017, Tokyo
		</p></td>

</tr>
  <tr>
    <td>
		<p>
			<a href="https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/slides/pg10-partition-pgcon-2017.pdf">
				<b>Partition and Conquer Large Data In PostgreSQL 10</b>
			</a>
		</p>
	
		<td><p>
			PGCon 2017, Ottawa
		</p></td>
	</td>

</tr>
  <tr>
    <td>
		<p>
			<a href="https://www.pgcon.org/2017/schedule/events/1069.en.html">
				<b>Towards Built-in Sharding in Community PostgreSQL (No slides!)</b>
			</a>
		</p>
	</td>
	
		<td><p>
			PGCon 2017, Ottawa
		</p></td>

</tr>
  <tr>
    <td>
		<p>
			<a href="https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/slides/pg10-features-ntt-techconf.pdf">
				<b>PostgreSQL 10: What to Look For</b>
			</a>
		</p>
	</td>
	
		<td><p>
			NTT Tech Conf #1, Tokyo
		</p></td>

</tr>
</table>
