---
title: "Your Bitcoin Payment Node"
layout: splash

date: 2016-03-23T11:48:41-04:00
header:
  overlay_color: "#000"
  overlay_filter: "0.3"
  overlay_image: /assets/img_unsplash_lightning_leon_contreras_6000x4005.jpg
  actions:
    - label: "Configure"
      url: "configure_node"
  caption: "Photo credit: [**Unsplash**](https://unsplash.com)"

# https://twitter.com/michael_saylor/status/1421806659039637508
excerpt: "The ultimate self-hosting solution for your payments on a Raspberry Pi. Receive payments in Bitcoin directly, privately and without fees!"

intro: 
  - excerpt: 'A node is an altar to truth. A wallet is a weapon of freedom. A miner is a motor of sovereignty. (Michael Saylor, 2021)'

feature_row:
  - image_path: /assets/index_support_for_bitcoin_and_lightning.svg
    alt: "Support for bitcoin and lightning"
    title: "Bitcoin and Lightning"
    excerpt: "Easily send and receive bitcoin in all its flavours."

  - image_path: /assets/index_easy_to_setup_and_configure.svg
    # image_caption: "Image courtesy of [Unsplash](https://unsplash.com/)"
    alt: "placeholder image 2"
    title: "Simple to configure"
    excerpt: "In just five steps, with video tutorials"
    url: "configure_node"
    btn_label: "Configure"
    btn_class: "btn--primary"

  - image_path: /assets/index_secured_full_disk_encryption.svg
    title: "Secure"
    excerpt: "Featuring full disk encryption, encrypted backups, two factor authentication and onion services."

feature_row_btcpayserver:
  - image_path: /assets/index_btcpay_logo_black_txt.svg
    alt: "placeholder image 2"
    title: "Powered by the awesome btcpayserver..."
    excerpt: 'BTCPay Server is a self-hosted, open-source cryptocurrency payment processor. Read more on their website.'
    url: "https://btcpayserver.org"
    btn_label: "Read More"
    btn_class: "btn--primary"

feature_row_apps:
  - image_path: /assets/index_extended_with_other_apps.png
    alt: "placeholder image 2"
    title: "...And extended with many other apps"
    excerpt: 'Including electrum server (powered by electrs) and electrum wallet, pihole, remote desktop web access (powered by guacamole).'

feature_row_opensource:
  - image_path: /assets/img_unsplash_source_2880x1920.jpg
    alt: "placeholder image 2"
    title: "Fully open source"
    excerpt: 'Checkout and improve what is installed on your device.'
    url: "https://github.com/gradientskier/btcpayserver-docker"
    btn_label: "Get the source code"
    btn_class: "btn--primary"
---

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

{% include feature_row id="feature_row_btcpayserver" type="left" %}

{% include feature_row id="feature_row_apps" type="right" %}

{% include feature_row id="feature_row_opensource" type="center" %}