jabas.experiments
===============

Welcome! This is my little GitHub corner to play with ideas, share snippets of code. Right now there's only one experiment uploaded, but I've got a few more ideas spinning around in my head.

# Experiment 1: Flexible Billboard
Within the site I'm currently responsible for, nearly every page begins with some sort of billboard-type banner, with many different variables.

- there's a thin, image only version that gets you right to the main body content
- there's a headline, catch-phrase version that can either be short or tall
- there's a more verbose version, with a headline, a paragraph of content and usually a CTA as a button or video. This one is always tall.

Complicating things, the responsive behavior of the tall version version is that it maintains it's aspect ratio between 887px and 1400px, where it transitions from 380px to 600px of height.

At the time this was developed, we were using Middleman to build a static version of our site, and the different variations were triggered by a frontmatter variable that added classes to our main body container adding padding-top to it, and using the absolute positioning, padding-bottom trick (a la <http://alistapart.com/article/creating-intrinsic-ratios-for-video>) on the banner container for the responsive changes.

The end result of what is detailed here is available at <http://jabas.netlify.com> and all of the files are within this GitHub repo.

## Mission 1
My first goal was to create a Middleman partial that would accept local variables to create all of the variations depending on what specific variables the developer added, without the developer having to declare a type. The least complicated usage would be:

```<%= partial "partials/billboard" %>```

...which would add a thin banner with a default image. But as the developer stacked more and more variables, the partial would add the containers and classes it needed to transform it to whatever version was appropriate.

### The Types
As described above, this broke out to three types of billboard:

- "Textless," which is always the thin version
- "Simple," short, with only room for a headline
- "Verbose," which was taller to accomodate the extra text and CTAs

### The Variables

- `background` - Needed to add a specific background beyond the default
- `headline` - If this was added, this was automatically "Simple" or "Verbose"
- `use_tall` - If the only content was a headline, but you still wanted the "Verbose" height, the only way to trigger that was with an extra Boolean.
- `body` - An accompanying paragraph of text. Once this or the following content fields were included the billboard was automatically "Verbose."
- `cta_type` - This could be either 'button' or 'video' to distinguish between a regular textual button or a video play icon.
- `btn_txt, btn_url, video_url` - These fill in the gaps on whatever CTA type is chosen (our site actually uses a more complicated system to get Vimeo to play via an iframe in a lightbox, but it's not helpful in this demonstration.)

## Mission 2

Next on the list, CSS styling. It was important to me that the new version be completely self-contained, so that this could be plugged in anywhere. 

More importantly the absolutely positioned solution to the responsive behavior in the old version often put us in the position of having too-long content push outside the bounds available to it. I needed to rework it so that it adhered to the old standards, but when content got too long it would push the height along with it, never appearing broken.

### Vertical Centering to the Rescue

After recieving countless designs that asked for vertically-centered content, I was used to the solution in <https://css-tricks.com/centering-in-the-unknown/> which uses an inline-block psuedo-element alongside an inline-block content container to vertcally-align. Our original version of the billboard looked like this (I'm leaving out styling not related to layout, and the aspects that fixed it to the top of the page):

```
.banner-content {
	position: relative;
	padding-bottom: 42.857%;
}
  
.billboard {
	position: absolute;
	top: 0; left: 0; right: 0; bottom: 0;
	overflow: hidden;

	&:before,
	.billboard-content {
		display: inline-block;
		vertical-align: middle;
	}

	&:before {
		content: "";
		height: 100%;
		margin-right: -0.35em;
	}

	.billboard-content {
		width: 99%;
	}
}
```

My "a-ha" moment was when I realized that because the inline-block will add padding-bottom based on the width of its parent container, I could use the centering psuedo-element to also enforce a percentage-based height as the tallest element in the line. If the content area became taller, it would be the basis of the height. Plus there was the bonus of no longer needing any absolute positioning. I've also since found I prefer the font-size: 0 method of removing inline-block gaps. The code:

```
.billboard {
	font-size: 0;

	&:before,
	.bb_content {
		display: inline-block;
		vertical-align: middle;
	}

	&:before {
		content: "";
		padding-bottom: 42.857%;
	}

	.bb_content {
		font-size: 16px;
	}
}
```

### Background Images
The original version of this required us to define the background-image and its responsive changes on the billboard within page-specific styles. With the new system, I wanted stakeholders to choose from an approved library of images to retain control. More importantly, since the height of the banner could change as the content changed, I wanted that change to happen automatically; i.e. if the banner went from simple to verbose, I wanted the correct image to be placed without developer intervention. Thus a new class-based system for applying new backgrounds:
```
.bg_default.bb_textless {
	background-image: url("/img/backgrounds/default-textless-768.jpg");
}
.bg_default.bb_simple {
	background-image: url("/img/backgrounds/default-simple-768.jpg");
}
.bg_default.bb_verbose {
	background-image: url("/img/backgrounds/default-verbose-768.jpg"); 
}
@media all and (min-width: 768px) {
	.bg_default.bb_textless {
		background-image: url("/img/backgrounds/default-textless-1600.jpg"); 
	}
	.bg_default.bb_simple {
		background-image: url("/img/backgrounds/default-simple-1600.jpg");
	}
	.bg_default.bb_verbose {
		background-image: url("/img/backgrounds/default-verbose-1600.jpg");
	}
}
```
## Mission 3: 

The last portion to this, and this was really more of a learning experience for me, was that I wanted to hook up this site with [Contentful](https://www.contentful.com/) to allow it to be edited by anyone with access remotely. I won't delve too much into the technicals of integration here, but the basic version is that my [Netlify](https://www.netlify.com) host allows for a webhook that updates the data and triggers a Middleman build. When Contentful data is published, it sends out a request to the webhook URL, kicking off the process.

Within Middleman, a Contentful plugin uses their API to pull in data I specify via ID, and write it into YAML data. The YAML is then used in the build itself. The billboard partial was updated to accept the data under the variable of `contentful`, after which it is translated into the data the partial already accepted. Code below:

```
if locals.has_key?(:contentful)
		
	background = contentful.backgroundImage
	headline = contentful.header || nil
	use_tall = contentful.useTallBillboard || false
	body = contentful.body || nil
	cta_type = contentful.ctaType || "none"
	btn_txt = contentful.buttonText || nil
	btn_url = contentful.buttonUrl || nil
	video_url = contentful.videoUrl || nil
```
## Conclusion
The code for the ERB partial can be viewed in its entirety at <https://github.com/jabas/jabas.experiments/blob/master/source/partials/_billboard.erb>. The accompanying CSS is at <https://github.com/jabas/jabas.experiments/blob/master/source/css/_billboard.scss> (note, this was stripped out of a fuller library of styles, so a large portion of it is adding back in styles and sizing for the content that would otherwise be housed elsewhere (and rem based sizing).

The only page in the Middleman build (currently) is at <https://github.com/jabas/jabas.experiments/blob/master/source/index.html.erb> and contains only examples of how different variable combinations give different results. The hosted version of that page is at <http://jabas.netlify.com/>

Thanks, and enjoy!
