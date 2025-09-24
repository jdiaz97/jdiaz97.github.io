+++
authors = ["José Díaz"]
title = "Using Rust to run the most powerful AI models for Camera Trap processing"
description = "Tiny code overviews on running computer vision models"
date = 2025-09-24
[taxonomies]
tags = ["Rust", "AI", "Computer vision", "Biodiversity"]
[extra]
banner = "banner.webp"
toc = true
toc_inline = true
toc_ordered = true
+++

An example of how AI is doing great things for the planet. This is about [BoquilaHUB](https://github.com/boquila/boquilahub/) and how it's helping a lot of people protect nature.

# Camera traps

The most important tool to monitor wildlife worldwide is camera traps. Every national park uses them.

The problem is that this creates a lot of data, millions of images that have to be processed somehow. The most recent and most powerful models that we can use to automate these tasks are the following:

# SpeciesNet and MegaDetector

Released just a few months ago, we have these two models:

- SpeciesNet, by Google, a classification EfficientNetV2 trained on millions of images with over 2000 classes. 55.9M parameters.  
- MDv-1000 (the latest in the MegaDetector series), which is an object detection model with 3 classes: animal, person, vehicle. 140.3M parameters in its larger version.

So, we can use MegaDetector to detect animals and SpeciesNet to get close to a species classification. Great! We can get close to automating camera trap processing.

# Processing

With a library, AI is very simple to run. The process is:  

Input Tensor -> Run inference -> Output Tensor -> A data representation that's useful for us.  

So let's do that. We'll be using [ort](https://github.com/pykeio/ort), version 2.0.0-rc.9. The API is slightly different in the rc.10 version, but the core logic is the same.

## Preprocessing

Getting an Input Tensor:

- For SpeciesNet, the input tensor needs to have the shape: NCHW.  
- For MDv-1000, the input tensor needs to have the shape: NHWC.  

Let's assume we have an image loaded in memory already, and we need to transform it.

```rust
pub enum TensorFormat {
    NCHW, // Batch, Channel, Height, Width
    NHWC, // Batch, Height, Width, Channel
}

pub fn imgbuf_to_input_array(
    batch_size: usize,
    input_depth: usize,
    input_height: u32,
    input_width: u32,
    img: &ImageBuffer<Rgb<u8>, Vec<u8>>,
    format: TensorFormat,
) -> (Array<f32, Ix4>, u32, u32) {
    let (img_width, img_height) = img.dimensions();

    let resized = fast_resize(img, input_width, input_height);

    let (h, w) = (input_height as usize, input_width as usize);
    let mut input = match format {
        TensorFormat::NCHW => Array::zeros((batch_size, input_depth, h, w)),
        TensorFormat::NHWC => Array::zeros((batch_size, h, w, input_depth)),
    };

    for (x, y, pixel) in resized.enumerate_pixels() {
        let (x, y) = (x as usize, y as usize);
        let [r, g, b, ..] = pixel.0;
        let rgb: [f32; 3] = [r as f32 * SCALE, g as f32 * SCALE, b as f32 * SCALE];

        match format {
            TensorFormat::NCHW => {
                input[[0, 0, y, x]] = rgb[0];
                input[[0, 1, y, x]] = rgb[1];
                input[[0, 2, y, x]] = rgb[2];
            }
            TensorFormat::NHWC => {
                input[[0, y, x, 0]] = rgb[0];
                input[[0, y, x, 1]] = rgb[1];
                input[[0, y, x, 2]] = rgb[2];
            }
        }
    }
    (input, img_width, img_height)
}
```

And that's it. From an image, we can create input tensors that work for both SpeciesNet and MegaDetector.

## Inference

```rust
pub fn inference<'a>(
    session: &'a ort::session::Session,
    input: &'a Array<f32, Ix4>,
    input_name: &str,
) -> ort::session::SessionOutputs<'a, 'a> {
    return session
        .run(ort::inputs![input_name => input.view()].unwrap())
        .unwrap();
}
```

## Post-processing

First, we extract the output tensor into an ndarray:

```rust
pub fn extract_output(
    outputs: &SessionOutputs<'_, '_>,
    output_name: &str,
) -> ArrayBase<OwnedRepr<f32>, Dim<IxDynImpl>> {
    return outputs[output_name]
        .try_extract_tensor::<f32>()
        .unwrap()
        .t()
        .into_owned();
}
```

Now we can process it and create a representation that's useful for us.

For SpeciesNet:

```rust
pub struct ProbSpace {
    pub classes: Vec<String>,
    pub probs: Vec<f32>,
    pub classes_ids: Vec<u32>,
}

pub fn process_class_output(
    conf: f32,
    classes: &Vec<String>,
    output: &Array<f32, IxDyn>,
) -> ProbSpace {
    let mut indexed_scores: Vec<(usize, f32)> = output
        .iter()
        .enumerate()
        .filter(|(_, &score)| score >= conf)
        .map(|(i, &score)| (i, score))
        .collect();

    indexed_scores.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

    let probs: Vec<f32> = indexed_scores.iter().map(|(_, prob)| *prob).collect();
    let classes_ids: Vec<u32> = indexed_scores.iter().map(|(idx, _)| *idx as u32).collect();
    let classes: Vec<String> = classes_ids
        .iter()
        .map(|&idx| classes[idx as usize].clone())
        .collect();

    return ProbSpace::new(classes, probs, classes_ids);
}
```

For MDv-1000:

```rust
#[derive(Serialize, Deserialize, Copy, Clone, Debug)]
pub struct XYXY {
    pub x1: f32,
    pub y1: f32,
    pub x2: f32,
    pub y2: f32,
    pub prob: f32,
    pub class_id: u32,
}

#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct XYXYc {
    pub xyxy: XYXY,
    pub label: String,
    pub extra_cls: Option<ProbSpace>, // If we want to run a classification model for the bounding box, we will save the results here.
}

impl MegaDetector {
    fn process_detect_output(
        &self,
        output: &Array<f32, IxDyn>,
        img_width: u32,
        img_height: u32,
    ) -> Vec<XYXYc> {
        let output = output.slice(s![.., .., 0]);
        let x_scale = img_width as f32 / self.input_width as f32;
        let y_scale = img_height as f32 / self.input_height as f32;

        let mut boxes: Vec<XYXY> = output
            .axis_iter(Axis(0))
            .filter_map(|row| {
                let (class_id, prob) = row
                    .iter()
                    .skip(4)
                    .enumerate()
                    .map(|(index, &value)| (index, value))
                    .reduce(|a, b| if b.1 > a.1 { b } else { a })?;

                if prob < self.config.confidence_threshold {
                    return None;
                }

                let xc = row[0 as usize] * x_scale;
                let yc = row[1 as usize] * y_scale;
                let w = row[2 as usize] * x_scale;
                let h = row[3 as usize] * y_scale;

                let x1 = xc - w * 0.5;
                let x2 = xc + w * 0.5;
                let y1 = yc - h * 0.5;
                let y2 = yc + h * 0.5;

                Some(XYXY::new(x1, y1, x2, y2, prob, class_id as u32))
            })
            .collect();

        for technique in &self.post_processing { 
            if matches!(technique, PostProcessing::NMS) {
                let indices = nms_indices(&boxes, self.config.nms_threshold);
                boxes = indices.iter().map(|&idx| boxes[idx]).collect();
            }
        }

        self.t(&boxes) // Vec<XYXY> to Vec<XYXYc>, which means just adding the String for the class.
    }
}
```

## Combining both models

The rest is just hacky work, you can imagine it very easily. For each bounding box:  

- Slice the image.  
- Run the extra classification model (SpeciesNet).  
- Add the predictions to the XYXYc struct in the extra_cls field. That's it.  

## Things I didn't mention

There is more post-processing involved. For example:

- "SpeciesNet thinks the animal is a kangaroo, but my data is not from Australia". Ok, we can use your location and "roll up" the classification up in the taxonomy level and weight the other predictions until we have a good conclusion.

That's just hacky work, useful but not interesting to read.

# Conclusion

I think Rust makes it easy to do this. Because:

- Enums are great for handling different cases in a simple way. In the actual code, there is more of that.
- The integration between ndarray and ort is very clean.
- Ndarrays are great to manage. Coming from Julia and Python I'm very satisfied with using ndarray.

# FAQ

Q: Bro, just deploy a Python inference endpoint using this cloud provider to avoid writing new code.  

A: Offline inference is very important for people working in national parks. Also, it would probably be very expensive if you're producing a lot of data, and everything related to biodiversity protection is underfunded.

Q: hey but ORT is just a wrapper for ONNX

A: Yeah but it's way cleaner than using ONNX from C. Also, they're trying to turn ORT into frontend for other frameworks, like [Candle](https://github.com/huggingface/candle). 
