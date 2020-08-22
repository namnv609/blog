---
layout: default
title: Homepage
---

<!-- Start latest-post Area -->
<div class="latest-post-wrap">
  <h4 class="cat-title">Latest News</h4>

  {% for post in site.posts %}
    <div class="single-latest-post row align-items-center">
      <div class="col-lg-5 post-left">
        <div class="feature-img relative">
          <div class="overlay overlay-bg"></div>
          <img class="img-fluid" src="{{ post.feature_image }}" alt="">
        </div>
        <ul class="tags">
          <li><a href="#">{{ post.category }}</a></li>
        </ul>
      </div>
      <div class="col-lg-7 post-right">
        <a href="{{ post.url }}">
          <h4>{{ post.title }}</h4>
        </a>
        <ul class="meta">
          <li><a href="#"><span class="lnr lnr-user"></span>{{ post.author }}</a></li>
          <li>
            <a href="#">
              <span class="lnr lnr-calendar-full"></span>
              {{ post.date | date: "%d %b, %Y" }}
            </a>
          </li>
        </ul>
        <p class="excert">
          {{ post.excerpt }}
        </p>
      </div>
    </div>
  {% endfor %}
</div>
<!-- End latest-post Area -->
