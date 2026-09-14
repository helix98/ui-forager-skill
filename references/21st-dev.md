# 21st.dev Collected Components

Personal stash of components manually pulled from 21st.dev (2-fetch/day limit there, so we save real fetches into this file instead of re-fetching). Check this file first before ever suggesting a live 21st.dev fetch.

---

## Reveal Wave Image
- **Category**: image effect / interactive canvas / hover reveal
- **Stack**: React + Three.js (`@react-three/fiber`, `@react-three/drei`), TypeScript, GLSL shaders
- **Deps**: `three`, `@react-three/fiber`, `@react-three/drei`
- **Summary**: Renders an image on a WebGL plane, displayed by default as animated black-and-white with a Bayer 4x4 dithered look and continuous wave distortion. On mouse hover it reveals full color in a soft-edged "flashlight" radius around the cursor and adds mouse-driven ripple distortion. Fades smoothly in/out as the mouse enters/leaves. Uses `object-fit: cover`-style responsive sizing via aspect-ratio-aware plane scaling.
- **Notes**: Heavier than a CSS-only effect — pulls in the full R3F/Three stack, so only reach for it when the animated dither+ripple+reveal look is actually wanted, not for a plain hover-zoom (use a lighter CSS/Motion approach for that). Configurable via props: `revealRadius`, `revealSoftness`, `pixelSize` (dither block size), `waveSpeed`, `waveFrequency`, `waveAmplitude`, `mouseRadius`.

```tsx
"use client";

import * as THREE from "three";
import { Canvas, useFrame, useThree } from "@react-three/fiber";
import { useTexture } from "@react-three/drei";
import { useMemo, useRef, useState, useEffect } from "react";

const vertexShader = `
  varying vec2 vUv;
  
  void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`;

const fragmentShader = `
  precision highp float;
  
  uniform sampler2D uTexture;
  uniform float uTime;
  uniform vec2 uMouse;
  uniform float uRevealRadius;
  uniform float uRevealSoftness;
  uniform float uPixelSize;
  uniform float uMouseActive;
  
  uniform float uWaveSpeed;
  uniform float uWaveFrequency;
  uniform float uWaveAmplitude;
  uniform float uMouseRadius;
  
  varying vec2 vUv;
  
  // Bayer 4x4 dithering pattern
  float bayer4x4(vec2 pos) {
    int x = int(mod(pos.x, 4.0));
    int y = int(mod(pos.y, 4.0));
    int index = x + y * 4;
    
    float pattern[16];
    pattern[0] = 0.0;    pattern[1] = 8.0;    pattern[2] = 2.0;    pattern[3] = 10.0;
    pattern[4] = 12.0;   pattern[5] = 4.0;    pattern[6] = 14.0;   pattern[7] = 6.0;
    pattern[8] = 3.0;    pattern[9] = 11.0;   pattern[10] = 1.0;   pattern[11] = 9.0;
    pattern[12] = 15.0;  pattern[13] = 7.0;   pattern[14] = 13.0;  pattern[15] = 5.0;
    
    for (int i = 0; i < 16; i++) {
        if (i == index) return pattern[i] / 16.0;
    }
    return 0.0;
  }
  
  void main() {
    vec2 uv = vUv;
    
    // Wave and Ripple Distortions
    float time = uTime;
    float waveStrength = uWaveAmplitude * 0.1;
    
    // Continuous waves
    float wave1 = sin(uv.y * uWaveFrequency + time * uWaveSpeed) * waveStrength;
    float wave2 = sin(uv.x * uWaveFrequency * 0.7 + time * uWaveSpeed * 0.8) * waveStrength * 0.5;
    
    vec2 distortedUv = uv;
    distortedUv.x += wave1;
    distortedUv.y += wave2;
    
    // Mouse interaction (Ripple)
    if (uMouseActive > 0.01) {
        vec2 mousePos = uMouse;
        float dist = distance(uv, mousePos);
        float mouseInfluence = smoothstep(uMouseRadius, 0.0, dist);
        
        float rippleFreq = uWaveFrequency * 5.0;
        float rippleSpeed = uWaveSpeed * 1.0;
        float rippleStrength = uWaveAmplitude * 0.05;
        
        float ripple = sin(dist * rippleFreq - time * rippleSpeed) * rippleStrength * mouseInfluence * uMouseActive;
        distortedUv.x += ripple;
        distortedUv.y += ripple;
    }
    
    // Sampling and Color Logic
    vec4 color = texture2D(uTexture, distortedUv);
    
    // Grayscale conversion
    float gray = dot(color.rgb, vec3(0.299, 0.587, 0.114));
    
    // Dithering
    vec2 pixelCoord = floor(gl_FragCoord.xy / uPixelSize);
    float dither = bayer4x4(pixelCoord);
    
    // 2-level quantization
    float quantized;
    float adjusted = gray + (dither - 0.5) * 0.5;
    if (adjusted < 0.33) {
        quantized = 0.0;
    } else if (adjusted < 0.66) {
        quantized = 0.5;
    } else {
        quantized = 1.0;
    }
    vec3 bwColor = vec3(quantized);
    
    // Reveal Flashlight
    float revealDist = distance(uv, uMouse);
    float innerRadius = uRevealRadius * (1.0 - uRevealSoftness);
    float outerRadius = uRevealRadius;
    float revealAmount = 1.0 - smoothstep(innerRadius, outerRadius, revealDist);
    revealAmount *= uMouseActive;
    
    vec3 finalColor = mix(bwColor, color.rgb, revealAmount);
    
    gl_FragColor = vec4(finalColor, color.a);
  }
`;

interface ImagePlaneProps {
  src: string;
  aspectRatio: number;
  revealRadius: number;
  revealSoftness: number;
  pixelSize: number;
  waveSpeed: number;
  waveFrequency: number;
  waveAmplitude: number;
  mouseRadius: number;
  isMouseInCanvas: boolean;
}

function ImagePlane({
  src,
  aspectRatio,
  revealRadius,
  revealSoftness,
  pixelSize,
  waveSpeed,
  waveFrequency,
  waveAmplitude,
  mouseRadius,
  isMouseInCanvas,
}: ImagePlaneProps) {
  const texture = useTexture(src);
  const meshRef = useRef<THREE.Mesh>(null);
  const { pointer } = useThree();
  const mouseActiveRef = useRef(0);
  const hasEnteredRef = useRef(false);

  const uniforms = useMemo(
    () => ({
      uTexture: { value: texture },
      uTime: { value: 0 },
      uMouse: { value: new THREE.Vector2(-10, -10) },
      uRevealRadius: { value: revealRadius },
      uRevealSoftness: { value: revealSoftness },
      uPixelSize: { value: pixelSize },
      uMouseActive: { value: 0 },
      uWaveSpeed: { value: waveSpeed },
      uWaveFrequency: { value: waveFrequency },
      uWaveAmplitude: { value: waveAmplitude },
      uMouseRadius: { value: mouseRadius },
    }),
    [
      texture,
      revealRadius,
      revealSoftness,
      pixelSize,
      waveSpeed,
      waveFrequency,
      waveAmplitude,
      mouseRadius,
    ],
  );

  const scale = useMemo<[number, number, number]>(() => {
    // Canvas clip-space is square (-1..1)
    if (aspectRatio > 1) {
      // Image is wider than tall
      return [aspectRatio, 1, 1];
    } else {
      // Image is taller than wide
      return [1, 1 / aspectRatio, 1];
    }
  }, [aspectRatio]);
  useFrame((state) => {
    if (meshRef.current) {
      const material = meshRef.current.material as THREE.ShaderMaterial;
      material.uniforms.uTime.value = state.clock.elapsedTime;

      if (isMouseInCanvas) {
        hasEnteredRef.current = true;
      }

      const targetActive = isMouseInCanvas ? 1 : 0;
      const easingSpeed = 0.08;
      mouseActiveRef.current +=
        (targetActive - mouseActiveRef.current) * easingSpeed;
      material.uniforms.uMouseActive.value = mouseActiveRef.current;

      if (hasEnteredRef.current) {
        material.uniforms.uMouse.value.set(
          (pointer.x + 1) / 2,
          (pointer.y + 1) / 2,
        );
      }
    }
  });

  return (
    <mesh ref={meshRef} scale={scale}>
      <planeGeometry args={[2, 2]} />
      <shaderMaterial
        vertexShader={vertexShader}
        fragmentShader={fragmentShader}
        uniforms={uniforms}
      />
    </mesh>
  );
}

interface RevealWaveImageProps {
  src: string;
  revealRadius?: number;
  revealSoftness?: number;
  pixelSize?: number;
  waveSpeed?: number;
  waveFrequency?: number;
  waveAmplitude?: number;
  mouseRadius?: number;
  className?: string;
}

export const RevealWaveImage = ({
  src,
  revealRadius = 0.2,
  revealSoftness = 0.5,
  pixelSize = 3,
  waveSpeed = 0.5,
  waveFrequency = 3.0,
  waveAmplitude = 0.2,
  mouseRadius = 0.2,
  className = "h-full w-full",
}: RevealWaveImageProps) => {
  const [isMouseInCanvas, setIsMouseInCanvas] = useState(false);
  const [aspectRatio, setAspectRatio] = useState<number | null>(null);

  useEffect(() => {
    const img = new Image();
    img.src = src;
    img.onload = () => {
      setAspectRatio(img.naturalWidth / img.naturalHeight);
    };
  }, [src]);

  return (
    <div
      className={`relative overflow-hidden ${className}`}
      onMouseEnter={() => setIsMouseInCanvas(true)}
      onMouseLeave={() => setIsMouseInCanvas(false)}
    >
      {aspectRatio !== null && (
        <Canvas
          style={{
            width: "100%",
            height: "100%",
            display: "block",
          }}
          gl={{ antialias: false }}
          camera={{ position: [0, 0, 1] }}
        >
          <ImagePlane
            src={src}
            aspectRatio={aspectRatio}
            revealRadius={revealRadius}
            revealSoftness={revealSoftness}
            pixelSize={pixelSize}
            waveSpeed={waveSpeed}
            waveFrequency={waveFrequency}
            waveAmplitude={waveAmplitude}
            mouseRadius={mouseRadius}
            isMouseInCanvas={isMouseInCanvas}
          />
        </Canvas>
      )}
    </div>
  );
}
```

**Demo usage:**
```tsx
import { RevealWaveImage } from "@/components/ui/reveal-wave-image";

export default function DemoOne() {
  return (
    <div className="w-screen h-svh">
      <RevealWaveImage
        src="https://images.unsplash.com/photo-1761839257469-96c78a7c2dd3"
        waveSpeed={0.2}
        waveFrequency={0.7}
        waveAmplitude={0.5}
        revealRadius={0.5}
        revealSoftness={1}
        pixelSize={2}
        mouseRadius={0.4}
      />
    </div>
  );
}
```

---

## Spotlight Card (GlowCard)
- **Category**: card / hover effect / cursor-reactive glow
- **Stack**: React, TypeScript, plain CSS (no external animation libs)
- **Deps**: none beyond React (uses inline `<style>` injection + CSS custom properties, no Tailwind config changes needed beyond utility classes already in use)
- **Summary**: A card wrapper that tracks the global pointer position and drives a CSS-variable-based radial-gradient "spotlight" — both a soft background glow and a bright border highlight — that follows the cursor anywhere on the page (listens on `document`, not just the card). Color is set via a `glowColor` prop (`blue`/`purple`/`green`/`red`/`orange`) mapped to a hue, or fully custom sizing via `width`/`height`/`customSize`.
- **Notes**: Listens on `document`-level `pointermove`, so the glow direction is based on cursor position relative to the whole viewport, not just hover-within-card — multiple cards on a page will all point toward the same cursor, which reads as a "shared light source" effect rather than independent per-card spotlights. If a self-contained per-card spotlight is wanted instead, swap the listener to the card element itself. Uses `dangerouslySetInnerHTML` to inject a shared `<style>` block for the `::before`/`::after` pseudo-elements — fine as a single instance, but if used many times on one page, hoist that style block once rather than per-card to avoid duplicate `<style>` tags.

```tsx
import React, { useEffect, useRef, ReactNode } from 'react';

interface GlowCardProps {
  children: ReactNode;
  className?: string;
  glowColor?: 'blue' | 'purple' | 'green' | 'red' | 'orange';
  size?: 'sm' | 'md' | 'lg';
  width?: string | number;
  height?: string | number;
  customSize?: boolean; // When true, ignores size prop and uses width/height or className
}

const glowColorMap = {
  blue: { base: 220, spread: 200 },
  purple: { base: 280, spread: 300 },
  green: { base: 120, spread: 200 },
  red: { base: 0, spread: 200 },
  orange: { base: 30, spread: 200 }
};

const sizeMap = {
  sm: 'w-48 h-64',
  md: 'w-64 h-80',
  lg: 'w-80 h-96'
};

const GlowCard: React.FC<GlowCardProps> = ({ 
  children, 
  className = '', 
  glowColor = 'blue',
  size = 'md',
  width,
  height,
  customSize = false
}) => {
  const cardRef = useRef<HTMLDivElement>(null);
  const innerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const syncPointer = (e: PointerEvent) => {
      const { clientX: x, clientY: y } = e;
      
      if (cardRef.current) {
        cardRef.current.style.setProperty('--x', x.toFixed(2));
        cardRef.current.style.setProperty('--xp', (x / window.innerWidth).toFixed(2));
        cardRef.current.style.setProperty('--y', y.toFixed(2));
        cardRef.current.style.setProperty('--yp', (y / window.innerHeight).toFixed(2));
      }
    };

    document.addEventListener('pointermove', syncPointer);
    return () => document.removeEventListener('pointermove', syncPointer);
  }, []);

  const { base, spread } = glowColorMap[glowColor];

  // Determine sizing
  const getSizeClasses = () => {
    if (customSize) {
      return ''; // Let className or inline styles handle sizing
    }
    return sizeMap[size];
  };

  const getInlineStyles = () => {
    const baseStyles = {
      '--base': base,
      '--spread': spread,
      '--radius': '14',
      '--border': '3',
      '--backdrop': 'hsl(0 0% 60% / 0.12)',
      '--backup-border': 'var(--backdrop)',
      '--size': '200',
      '--outer': '1',
      '--border-size': 'calc(var(--border, 2) * 1px)',
      '--spotlight-size': 'calc(var(--size, 150) * 1px)',
      '--hue': 'calc(var(--base) + (var(--xp, 0) * var(--spread, 0)))',
      backgroundImage: `radial-gradient(
        var(--spotlight-size) var(--spotlight-size) at
        calc(var(--x, 0) * 1px)
        calc(var(--y, 0) * 1px),
        hsl(var(--hue, 210) calc(var(--saturation, 100) * 1%) calc(var(--lightness, 70) * 1%) / var(--bg-spot-opacity, 0.1)), transparent
      )`,
      backgroundColor: 'var(--backdrop, transparent)',
      backgroundSize: 'calc(100% + (2 * var(--border-size))) calc(100% + (2 * var(--border-size)))',
      backgroundPosition: '50% 50%',
      backgroundAttachment: 'fixed',
      border: 'var(--border-size) solid var(--backup-border)',
      position: 'relative' as const,
      touchAction: 'none' as const,
    };

    // Add width and height if provided
    if (width !== undefined) {
      baseStyles.width = typeof width === 'number' ? `${width}px` : width;
    }
    if (height !== undefined) {
      baseStyles.height = typeof height === 'number' ? `${height}px` : height;
    }

    return baseStyles;
  };

  const beforeAfterStyles = `
    [data-glow]::before,
    [data-glow]::after {
      pointer-events: none;
      content: "";
      position: absolute;
      inset: calc(var(--border-size) * -1);
      border: var(--border-size) solid transparent;
      border-radius: calc(var(--radius) * 1px);
      background-attachment: fixed;
      background-size: calc(100% + (2 * var(--border-size))) calc(100% + (2 * var(--border-size)));
      background-repeat: no-repeat;
      background-position: 50% 50%;
      mask: linear-gradient(transparent, transparent), linear-gradient(white, white);
      mask-clip: padding-box, border-box;
      mask-composite: intersect;
    }
    
    [data-glow]::before {
      background-image: radial-gradient(
        calc(var(--spotlight-size) * 0.75) calc(var(--spotlight-size) * 0.75) at
        calc(var(--x, 0) * 1px)
        calc(var(--y, 0) * 1px),
        hsl(var(--hue, 210) calc(var(--saturation, 100) * 1%) calc(var(--lightness, 50) * 1%) / var(--border-spot-opacity, 1)), transparent 100%
      );
      filter: brightness(2);
    }
    
    [data-glow]::after {
      background-image: radial-gradient(
        calc(var(--spotlight-size) * 0.5) calc(var(--spotlight-size) * 0.5) at
        calc(var(--x, 0) * 1px)
        calc(var(--y, 0) * 1px),
        hsl(0 100% 100% / var(--border-light-opacity, 1)), transparent 100%
      );
    }
    
    [data-glow] [data-glow] {
      position: absolute;
      inset: 0;
      will-change: filter;
      opacity: var(--outer, 1);
      border-radius: calc(var(--radius) * 1px);
      border-width: calc(var(--border-size) * 20);
      filter: blur(calc(var(--border-size) * 10));
      background: none;
      pointer-events: none;
      border: none;
    }
    
    [data-glow] > [data-glow]::before {
      inset: -10px;
      border-width: 10px;
    }
  `;

  return (
    <>
      <style dangerouslySetInnerHTML={{ __html: beforeAfterStyles }} />
      <div
        ref={cardRef}
        data-glow
        style={getInlineStyles()}
        className={`
          ${getSizeClasses()}
          ${!customSize ? 'aspect-[3/4]' : ''}
          rounded-2xl 
          relative 
          grid 
          grid-rows-[1fr_auto] 
          shadow-[0_1rem_2rem_-1rem_black] 
          p-4 
          gap-4 
          backdrop-blur-[5px]
          ${className}
        `}
      >
        <div ref={innerRef} data-glow></div>
        {children}
      </div>
    </>
  );
};

export { GlowCard }
```

**Demo usage:**
```tsx
import { GlowCard } from "@/components/ui/spotlight-card";

export function Default(){
  return(
    <div className="w-screen h-screen flex flex-row items-center justify-center gap-10 custom-cursor">
      <GlowCard />
      <GlowCard />
      <GlowCard />
    </div>
  );
};
```

---

## Liquid Glass
- **Category**: glassmorphism effect / dock / button — visual style primitive
- **Stack**: React, TypeScript, Tailwind utility classes + inline styles, raw SVG filter (no external deps)
- **Deps**: none beyond React — the distortion effect is a plain SVG `<filter>` (feTurbulence + feDisplacementMap + feSpecularLighting), applied via CSS `filter: url(#glass-distortion)`
- **Summary**: Apple "Liquid Glass"-style frosted, refractive panel effect. A reusable `GlassEffect` wrapper stacks a blurred/distorted backdrop layer, a translucent white tint layer, and an inset-highlight layer to fake light bending through glass, then renders children on top. Built on top of it: `GlassDock` (icon dock, hover-scales icons) and `GlassButton` (pill button with hover grow + press-scale). Requires the `GlassFilter` SVG (`id="glass-distortion"`) to be rendered once in the page/app for the distortion to apply.
- **Notes**: `GlassFilter`'s `<svg>` must be mounted somewhere in the tree (it's `display: none` but the filter def is referenced by id) — easy to forget and get a flat blur with no refraction if omitted. The wrapper hardcodes `rounded-3xl`/`rounded-4xl` and specific hover-grow paddings per variant (dock vs button); adjust those utility classes to match the project's actual radius/spacing scale rather than using as-is. `target="_blank"` is the `GlassEffect` default when `href` is passed — probably want to drop that for in-app links. Works best over a busy/colorful background (image, gradient) since the whole point is visible refraction — flat solid backgrounds won't show the effect much.

```tsx
"use client";

import React from "react";

// Types
interface GlassEffectProps {
  children: React.ReactNode;
  className?: string;
  style?: React.CSSProperties;
  href?: string;
  target?: string;
}

interface DockIcon {
  src: string;
  alt: string;
  onClick?: () => void;
}

// Glass Effect Wrapper Component
const GlassEffect: React.FC<GlassEffectProps> = ({
  children,
  className = "",
  style = {},
  href,
  target = "_blank",
}) => {
  const glassStyle = {
    boxShadow: "0 6px 6px rgba(0, 0, 0, 0.2), 0 0 20px rgba(0, 0, 0, 0.1)",
    transitionTimingFunction: "cubic-bezier(0.175, 0.885, 0.32, 2.2)",
    ...style,
  };

  const content = (
    <div
      className={`relative flex font-semibold overflow-hidden text-black cursor-pointer transition-all duration-700 ${className}`}
      style={glassStyle}
    >
      {/* Glass Layers */}
      <div
        className="absolute inset-0 z-0 overflow-hidden rounded-inherit rounded-3xl"
        style={{
          backdropFilter: "blur(3px)",
          filter: "url(#glass-distortion)",
          isolation: "isolate",
        }}
      />
      <div
        className="absolute inset-0 z-10 rounded-inherit"
        style={{ background: "rgba(255, 255, 255, 0.25)" }}
      />
      <div
        className="absolute inset-0 z-20 rounded-inherit rounded-3xl overflow-hidden"
        style={{
          boxShadow:
            "inset 2px 2px 1px 0 rgba(255, 255, 255, 0.5), inset -1px -1px 1px 1px rgba(255, 255, 255, 0.5)",
        }}
      />

      {/* Content */}
      <div className="relative z-30">{children}</div>
    </div>
  );

  return href ? (
    <a href={href} target={target} rel="noopener noreferrer" className="block">
      {content}
    </a>
  ) : (
    content
  );
};

// Dock Component
const GlassDock: React.FC<{ icons: DockIcon[]; href?: string }> = ({
  icons,
  href,
}) => (
  <GlassEffect
    href={href}
    className="rounded-3xl p-3 hover:p-4 hover:rounded-4xl"
  >
    <div className="flex items-center justify-center gap-2 rounded-3xl p-3 py-0 px-0.5 overflow-hidden">
      {icons.map((icon, index) => (
        <img
          key={index}
          src={icon.src}
          alt={icon.alt}
          className="w-16 h-16 transition-all duration-700 hover:scale-110 cursor-pointer"
          style={{
            transformOrigin: "center center",
            transitionTimingFunction: "cubic-bezier(0.175, 0.885, 0.32, 2.2)",
          }}
          onClick={icon.onClick}
        />
      ))}
    </div>
  </GlassEffect>
);

// Button Component
const GlassButton: React.FC<{ children: React.ReactNode; href?: string }> = ({
  children,
  href,
}) => (
  <GlassEffect
    href={href}
    className="rounded-3xl px-10 py-6 hover:px-11 hover:py-7 hover:rounded-4xl overflow-hidden"
  >
    <div
      className="transition-all duration-700 hover:scale-95"
      style={{
        transitionTimingFunction: "cubic-bezier(0.175, 0.885, 0.32, 2.2)",
      }}
    >
      {children}
    </div>
  </GlassEffect>
);

// SVG Filter Component
const GlassFilter: React.FC = () => (
  <svg style={{ display: "none" }}>
    <filter
      id="glass-distortion"
      x="0%"
      y="0%"
      width="100%"
      height="100%"
      filterUnits="objectBoundingBox"
    >
      <feTurbulence
        type="fractalNoise"
        baseFrequency="0.001 0.005"
        numOctaves="1"
        seed="17"
        result="turbulence"
      />
      <feComponentTransfer in="turbulence" result="mapped">
        <feFuncR type="gamma" amplitude="1" exponent="10" offset="0.5" />
        <feFuncG type="gamma" amplitude="0" exponent="1" offset="0" />
        <feFuncB type="gamma" amplitude="0" exponent="1" offset="0.5" />
      </feComponentTransfer>
      <feGaussianBlur in="turbulence" stdDeviation="3" result="softMap" />
      <feSpecularLighting
        in="softMap"
        surfaceScale="5"
        specularConstant="1"
        specularExponent="100"
        lightingColor="white"
        result="specLight"
      >
        <fePointLight x="-200" y="-200" z="300" />
      </feSpecularLighting>
      <feComposite
        in="specLight"
        operator="arithmetic"
        k1="0"
        k2="1"
        k3="1"
        k4="0"
        result="litImage"
      />
      <feDisplacementMap
        in="SourceGraphic"
        in2="softMap"
        scale="200"
        xChannelSelector="R"
        yChannelSelector="G"
      />
    </filter>
  </svg>
);

// Main Component
export const Component = () => {
  const dockIcons: DockIcon[] = [
    { src: "...", alt: "Claude" },
    { src: "...", alt: "Finder" },
    { src: "...", alt: "Chatgpt" },
    { src: "...", alt: "Maps" },
    { src: "...", alt: "Safari" },
    { src: "...", alt: "Steam" },
  ];

  return (
    <div
      className="min-h-screen h-full flex items-center justify-center font-light relative overflow-hidden w-full"
      style={{
        background: `url("...") center center`,
        animation: "moveBackground 60s linear infinite",
      }}
    >
      <GlassFilter />

      <div className="flex flex-col gap-6 items-center justify-center w-full">
        <GlassDock icons={dockIcons} href="https://example.com" />

        <GlassButton href="https://example.com">
          <div className="text-xl text-white">
            <p>How can i help you today?</p>
          </div>
        </GlassButton>
      </div>     
    </div>
  );
}
```

**Demo usage:**
```tsx
import { Component } from "@/components/ui/liquid-glass";

const DemoOne = () => {
  return <Component />;
};

export { DemoOne };
```

---

## Water Ripple Image
- **Category**: image effect / full-screen background / raw WebGL
- **Stack**: React, TypeScript, raw WebGL (no Three.js/R3F — hand-written GL context, shaders, program setup)
- **Deps**: none beyond React — everything is a plain `<canvas>` + WebGL calls, no `three`/`@react-three/*` needed
- **Summary**: Full-viewport animated water-surface distortion effect over an image, using layered simplex-noise-driven surface distortion + "outer noise" ripple to fake caustics/water refraction, plus an illumination pass that brightens/tints (default blueish) where the simulated surface catches light. Continuously animated via `requestAnimationFrame`, using `performance.now()` as the time uniform. Supports swapping the image at runtime via a hidden file input (`#image-selector-input`) that reads a local file and re-uploads it as the GL texture.
- **Notes**: Sizes the canvas to `window.innerWidth/innerHeight` (full viewport) rather than its container — it's built as a full-screen background piece, not a drop-in bounded image component like Reveal Wave Image is. To use it inside a normal-sized card/container, the resize logic needs to be changed to measure the container instead of `window`. Since it's raw WebGL (not R3F), it's lighter-weight dependency-wise than the Reveal Wave Image component, but any changes to the render loop/cleanup need to be done by hand rather than via React Three Fiber's abstractions. The unused `showControls` prop destructured in the demo's props type suggests the original had a debug UI panel that got stripped — fine to ignore or add back if runtime tweaking of `blueish`/`scale`/etc. is wanted.

```tsx
'use client';

import React, { useEffect, useMemo, useRef, useState } from 'react';

type Params = {
  blueish: number;
  scale: number;
  illumination: number;
  surfaceDistortion: number;
  waterDistortion: number;
  /** default image to load initially */
  src: string;
};

export type WaterRippleImageProps = Partial<Params> & {
  /** Extra class on the canvas wrapper */
  className?: string;
};

const VERT = `
precision mediump float;
varying vec2 vUv;
attribute vec2 a_position;
void main() {
  vUv = .5 * (a_position + 1.);
  gl_Position = vec4(a_position, 0.0, 1.0);
}
`;

const FRAG = `
precision mediump float;

varying vec2 vUv;
uniform sampler2D u_image_texture;
uniform float u_time;
uniform float u_ratio;
uniform float u_img_ratio;
uniform float u_blueish;
uniform float u_scale;
uniform float u_illumination;
uniform float u_surface_distortion;
uniform float u_water_distortion;

#define TWO_PI 6.28318530718
#define PI 3.14159265358979323846

vec3 mod289(vec3 x) { return x - floor(x * (1. / 289.)) * 289.; }
vec2 mod289(vec2 x) { return x - floor(x * (1. / 289.)) * 289.; }
vec3 permute(vec3 x) { return mod289(((x*34.)+1.)*x); }
float snoise(vec2 v) {
  const vec4 C = vec4(0.211324865405187, 0.366025403784439, -0.577350269189626, 0.024390243902439);
  vec2 i = floor(v + dot(v, C.yy));
  vec2 x0 = v - i + dot(i, C.xx);
  vec2 i1;
  i1 = (x0.x > x0.y) ? vec2(1., 0.) : vec2(0., 1.);
  vec4 x12 = x0.xyxy + C.xxzz;
  x12.xy -= i1;
  i = mod289(i);
  vec3 p = permute(permute(i.y + vec3(0., i1.y, 1.)) + i.x + vec3(0., i1.x, 1.));
  vec3 m = max(0.5 - vec3(dot(x0, x0), dot(x12.xy, x12.xy), dot(x12.zw, x12.zw)), 0.);
  m = m*m;
  m = m*m;
  vec3 x = 2. * fract(p * C.www) - 1.;
  vec3 h = abs(x) - 0.5;
  vec3 ox = floor(x + 0.5);
  vec3 a0 = x - ox;
  m *= 1.79284291400159 - 0.85373472095314 * (a0*a0 + h*h);
  vec3 g;
  g.x = a0.x * x0.x + h.x * x0.y;
  g.yz = a0.yz * x12.xz + h.yz * x12.yw;
  return 130. * dot(m, g);
}

mat2 rotate2D(float r) {
  return mat2(cos(r), sin(r), -sin(r), cos(r));
}

float surface_noise(vec2 uv, float t, float scale) {
  vec2 n = vec2(.1);
  vec2 N = vec2(.1);
  mat2 m = rotate2D(.5);
  for (int j = 0; j < 10; j++) {
    uv *= m;
    n *= m;
    vec2 q = uv * scale + float(j) + n + (.5 + .5 * float(j)) * (mod(float(j), 2.) - 1.) * t;
    n += sin(q);
    N += cos(q) / scale;
    scale *= 1.2;
  }
  return (N.x + N.y + .1);
}

void main() {
  vec2 uv = vUv;
  uv.y = 1. - uv.y;
  uv.x *= u_ratio;

  float t = .002 * u_time;
  vec3 color = vec3(0.);
  float opacity = 0.;

  float outer_noise = snoise((.3 + .1 * sin(t)) * uv + vec2(0., .2 * t));
  vec2 surface_noise_uv = 2. * uv + (outer_noise * .2);

  float surf = surface_noise(surface_noise_uv, t, u_scale);
  surf *= pow(uv.y, .3);
  surf = pow(surf, 2.);

  vec2 img_uv = vUv;
  img_uv -= .5;
  if (u_ratio > u_img_ratio) {
    img_uv.x = img_uv.x * u_ratio / u_img_ratio;
  } else {
    img_uv.y = img_uv.y * u_img_ratio / u_ratio;
  }
  float scale_factor = 1.4;
  img_uv *= scale_factor;
  img_uv += .5;
  img_uv.y = 1. - img_uv.y;

  img_uv += (u_water_distortion * outer_noise);
  img_uv += (u_surface_distortion * surf);

  vec4 img = texture2D(u_image_texture, img_uv);
  img *= (1. + u_illumination * surf);

  color += img.rgb;
  color += u_illumination * vec3(1. - u_blueish, 1., 1.) * surf;
  opacity += img.a;

  float edge_width = .02;
  float edge_alpha = smoothstep(0., edge_width, img_uv.x) * smoothstep(1., 1. - edge_width, img_uv.x);
  edge_alpha *= smoothstep(0., edge_width, img_uv.y) * smoothstep(1., 1. - edge_width, img_uv.y);
  color *= edge_alpha;
  opacity *= edge_alpha;

  gl_FragColor = vec4(color, opacity);
}
`;

function compileShader(gl: WebGLRenderingContext, src: string, type: number) {
  const sh = gl.createShader(type)!;
  gl.shaderSource(sh, src);
  gl.compileShader(sh);
  if (!gl.getShaderParameter(sh, gl.COMPILE_STATUS)) {
    const info = gl.getShaderInfoLog(sh);
    gl.deleteShader(sh);
    throw new Error(`Shader compile error: ${info || 'unknown'}`);
  }
  return sh;
}

function createProgram(gl: WebGLRenderingContext, vs: string, fs: string) {
  const v = compileShader(gl, vs, gl.VERTEX_SHADER);
  const f = compileShader(gl, fs, gl.FRAGMENT_SHADER);
  const prog = gl.createProgram()!;
  gl.attachShader(prog, v);
  gl.attachShader(prog, f);
  gl.linkProgram(prog);
  if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
    const info = gl.getProgramInfoLog(prog);
    gl.deleteProgram(prog);
    throw new Error(`Program link error: ${info || 'unknown'}`);
  }
  return prog;
}

export default function WaterRippleImage({
  blueish = 0.6,
  scale = 7,
  illumination = 0.15,
  surfaceDistortion = 0.07,
  waterDistortion = 0.03,
  src = 'https://images.unsplash.com/photo-1501785888041-af3ef285b470',
  showControls = true,
  className = '',
}: WaterRippleImageProps) {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const inputRef = useRef<HTMLInputElement | null>(null);

  const glRef = useRef<WebGLRenderingContext | null>(null);
  const programRef = useRef<WebGLProgram | null>(null);
  const uniformsRef = useRef<Record<string, WebGLUniformLocation | null>>({});
  const texRef = useRef<WebGLTexture | null>(null);
  const imgRef = useRef<HTMLImageElement | null>(null);
  const animRef = useRef<number | null>(null);

  const [params, setParams] = useState<Params>({
    blueish, scale, illumination, surfaceDistortion, waterDistortion, src,
  });

  const dpr = typeof window !== 'undefined' ? Math.min(window.devicePixelRatio || 1, 2) : 1;

  const updateUniforms = (gl: WebGLRenderingContext) => {
    const u = uniformsRef.current;
    gl.uniform1f(u['u_blueish'], params.blueish);
    gl.uniform1f(u['u_scale'], params.scale);
    gl.uniform1f(u['u_illumination'], params.illumination);
    gl.uniform1f(u['u_surface_distortion'], params.surfaceDistortion);
    gl.uniform1f(u['u_water_distortion'], params.waterDistortion);
  };

  const setTextureFromImage = (gl: WebGLRenderingContext, image: HTMLImageElement) => {
    if (texRef.current) gl.deleteTexture(texRef.current);
    const texture = gl.createTexture()!;
    texRef.current = texture;
    gl.activeTexture(gl.TEXTURE0);
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, 0);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, image);

    const u = uniformsRef.current;
    gl.uniform1i(u['u_image_texture'], 0);

    const imgRatio = image.naturalWidth / image.naturalHeight;
    const canvas = canvasRef.current!;
    gl.uniform1f(u['u_ratio'], canvas.width / canvas.height);
    gl.uniform1f(u['u_img_ratio'], imgRatio);
  };

  const loadImage = (srcUrl: string, gl: WebGLRenderingContext) =>
    new Promise<void>((resolve, reject) => {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.onload = () => {
        imgRef.current = img;
        setTextureFromImage(gl, img);
        resolve();
      };
      img.onerror = reject;
      img.src = srcUrl;
    });

  const resize = () => {
    const gl = glRef.current;
    const canvas = canvasRef.current;
    if (!gl || !canvas) return;

    const w = Math.floor(window.innerWidth * dpr);
    const h = Math.floor(window.innerHeight * dpr);
    if (canvas.width !== w || canvas.height !== h) {
      canvas.width = w;
      canvas.height = h;
    }
    gl.viewport(0, 0, canvas.width, canvas.height);

    const u = uniformsRef.current;
    gl.uniform1f(u['u_ratio'], canvas.width / canvas.height);

    if (imgRef.current) {
      const imgRatio = imgRef.current.naturalWidth / imgRef.current.naturalHeight;
      gl.uniform1f(u['u_img_ratio'], imgRatio);
    }
  };

  useEffect(() => {
    const canvas = canvasRef.current!;
    const gl =
      canvas.getContext('webgl', { alpha: true, antialias: true }) ||
      (canvas.getContext('experimental-webgl') as WebGLRenderingContext | null);

    if (!gl) {
      console.error('WebGL not supported');
      return;
    }
    glRef.current = gl;

    const program = createProgram(gl, VERT, FRAG);
    programRef.current = program;
    gl.useProgram(program);

    const uniformCount = gl.getProgramParameter(program, gl.ACTIVE_UNIFORMS);
    for (let i = 0; i < uniformCount; i++) {
      const info = gl.getActiveUniform(program, i);
      if (!info) continue;
      uniformsRef.current[info.name] = gl.getUniformLocation(program, info.name);
    }

    const vertices = new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1]);
    const vbo = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, vbo);
    gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

    const posLoc = gl.getAttribLocation(program, 'a_position');
    gl.enableVertexAttribArray(posLoc);
    gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 0, 0);

    updateUniforms(gl);
    loadImage(params.src, gl).catch((e) => console.error(e));

    resize();
    const onResize = () => resize();
    window.addEventListener('resize', onResize);

    const render = () => {
      const u = uniformsRef.current;
      if (u['u_time']) {
        gl.uniform1f(u['u_time'], performance.now());
      }
      gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
      animRef.current = requestAnimationFrame(render);
    };
    animRef.current = requestAnimationFrame(render);

    return () => {
      window.removeEventListener('resize', onResize);
      if (animRef.current) cancelAnimationFrame(animRef.current);
      if (texRef.current) gl.deleteTexture(texRef.current);
      gl.useProgram(null);
      if (programRef.current) gl.deleteProgram(programRef.current);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  useEffect(() => {
    const gl = glRef.current;
    if (!gl) return;
    updateUniforms(gl);
  }, [params.blueish, params.scale, params.illumination, params.surfaceDistortion, params.waterDistortion]);

  useEffect(() => {
    const input = inputRef.current;
    if (!input) return;

    const onChange = () => {
      const [file] = input.files ?? [];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = (e) => {
        const dataUrl = e.target?.result as string;
        const gl = glRef.current;
        if (!gl) return;
        loadImage(dataUrl, gl).catch((err) => console.error(err));
      };
      reader.readAsDataURL(file);
    };

    input.addEventListener('change', onChange);
    return () => input.removeEventListener('change', onChange);
  }, []);

  return (
    <div className={`relative w-screen h-screen ${className}`}>
      <input ref={inputRef} id="image-selector-input" type="file" className="hidden" />
      <canvas
        ref={canvasRef}
        className="fixed inset-0 w-screen"
        style={{ top: 0, left: 0 }}
      />
    </div>
  );
}

export { WaterRippleImage };
```

**Demo usage:**
```tsx
import { WaterRippleImage } from "@/components/ui/water-ripple-image";

export default function DemoOne() {
 return (
    <div className="min-h-screen">
      <WaterRippleImage
        blueish={0.4}
        scale={7}
        illumination={0.15}
        surfaceDistortion={0.03}
        waterDistortion={0.02}
        src="https://images.unsplash.com/photo-1501785888041-af3ef285b470"        
      />
    </div>
  );
}
```

---

## Testimonials with Marquee
- **Category**: marketing section / social proof / marquee layout
- **Stack**: React, Tailwind (shadcn `cn()` util), CSS animation (`animate-marquee` custom keyframe, no JS animation lib)
- **Deps**: `@/lib/utils` (shadcn's `cn`), and a separate `TestimonialCard` component + `TestimonialAuthor` type from `@/components/ui/testimonial-card` — **this component is not self-contained**, that card component wasn't included and needs to be fetched/written separately before this will work.
- **Summary**: A full testimonials section (heading + description + infinite horizontal marquee of quote cards). Duplicates the testimonials array 4x to make the loop seamless, animates via a CSS class `animate-marquee` (must be defined in Tailwind config/globals — not included here), pauses on hover via `group-hover:[animation-play-state:paused]`, and fades the edges with left/right gradient overlays so cards fade in/out at the section boundary instead of hard-cutting.
- **Notes**: Two things must exist elsewhere in the project for this to actually render: (1) the `animate-marquee` keyframe animation (define in `tailwind.config` as a keyframe moving `translateX(0)` → `translateX(calc(-100% - var(--gap)))`, or in global CSS), and (2) the `TestimonialCard`/`TestimonialAuthor` component itself. Ask the user for `TestimonialCard`'s code if they haven't provided it, or offer to build a simple one (avatar, name, handle, quote text, optional link wrapper) matching this component's expected props shape (`author: { name, handle, avatar }`, `text`, optional `href`). Gradient fade edges (`bg-gradient-to-r from-background`) assume a `background` CSS variable/token exists (standard in shadcn setups) — adjust to the project's actual token if not using shadcn's theme setup.

```tsx
import { cn } from "@/lib/utils"
import { TestimonialCard, TestimonialAuthor } from "@/components/ui/testimonial-card"

interface TestimonialsSectionProps {
  title: string
  description: string
  testimonials: Array<{
    author: TestimonialAuthor
    text: string
    href?: string
  }>
  className?: string
}

export function TestimonialsSection({ 
  title,
  description,
  testimonials,
  className 
}: TestimonialsSectionProps) {
  return (
    <section className={cn(
      "bg-background text-foreground",
      "py-12 sm:py-24 md:py-32 px-0",
      className
    )}>
      <div className="mx-auto flex max-w-container flex-col items-center gap-4 text-center sm:gap-16">
        <div className="flex flex-col items-center gap-4 px-4 sm:gap-8">
          <h2 className="max-w-[720px] text-3xl font-semibold leading-tight sm:text-5xl sm:leading-tight">
            {title}
          </h2>
          <p className="text-md max-w-[600px] font-medium text-muted-foreground sm:text-xl">
            {description}
          </p>
        </div>
        <div className="relative flex w-full flex-col items-center justify-center overflow-hidden">
          <div className="group flex overflow-hidden p-2 [--gap:1rem] [gap:var(--gap)] flex-row [--duration:40s]">
            <div className="flex shrink-0 justify-around [gap:var(--gap)] animate-marquee flex-row group-hover:[animation-play-state:paused]">
              {[...Array(4)].map((_, setIndex) => (
                testimonials.map((testimonial, i) => (
                  <TestimonialCard 
                    key={`${setIndex}-${i}`}
                    {...testimonial}
                  />
                ))
              ))}
            </div>
          </div>
          <div className="pointer-events-none absolute inset-y-0 left-0 hidden w-1/3 bg-gradient-to-r from-background sm:block" />
          <div className="pointer-events-none absolute inset-y-0 right-0 hidden w-1/3 bg-gradient-to-l from-background sm:block" />
        </div>
      </div>
    </section>
  )
}
```

**Demo usage:**
```tsx
import { TestimonialsSection } from "@/components/blocks/testimonials-with-marquee"

const testimonials = [
  {
    author: {
      name: "Emma Thompson",
      handle: "@emmaai",
      avatar: "https://images.unsplash.com/photo-1494790108377-be9c29b29330?w=150&h=150&fit=crop&crop=face"
    },
    text: "Using this AI platform has transformed how we handle data analysis. The speed and accuracy are unprecedented.",
    href: "https://twitter.com/emmaai"
  },
  {
    author: {
      name: "David Park",
      handle: "@davidtech",
      avatar: "https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?w=150&h=150&fit=crop&crop=face"
    },
    text: "The API integration is flawless. We've reduced our development time by 60% since implementing this solution.",
    href: "https://twitter.com/davidtech"
  },
  {
    author: {
      name: "Sofia Rodriguez",
      handle: "@sofiaml",
      avatar: "https://images.unsplash.com/photo-1534528741775-53994a69daeb?w=150&h=150&fit=crop&crop=face"
    },
    text: "Finally, an AI tool that actually understands context! The accuracy in natural language processing is impressive."
  }
]

export function TestimonialsSectionDemo() {
  return (
    <TestimonialsSection
      title="Trusted by developers worldwide"
      description="Join thousands of developers who are already building the future with our AI platform"
      testimonials={testimonials}
    />
  )
}
```

---

## Liquid Effect Animation
- **Category**: full-screen background effect / metallic liquid simulation
- **Stack**: React, TypeScript — but the actual effect is NOT self-contained code, it's a dynamically-injected `<script type="module">` that imports a third-party CDN package at runtime
- **Deps**: `threejs-components` pulled live from `https://cdn.jsdelivr.net/npm/threejs-components@0.0.22/build/backgrounds/liquid1.min.js` (loaded via a `<script>` tag injected into `document.body`, not an npm install)
- **Summary**: Renders a full-viewport metallic/liquid displacement effect (via the `threejs-components` `LiquidBackground` helper) driven by a loaded image, with configurable `metalness`/`roughness`/`displacementScale` and a `setRain` toggle. Cleans up by disposing the app and removing the injected script on unmount.
- **Notes — flag before using**: This is meaningfully different from the other collected components: instead of shipping the Three.js scene code, it loads a whole third-party library from a CDN at runtime via script injection, pinned to `@0.0.22` (an early/unstable-looking version). Before using this: (1) confirm the user is fine loading unaudited third-party code from a CDN at runtime in production — that's a real supply-chain consideration, not just a style choice; (2) suggest installing `threejs-components` as an actual npm dependency and importing it normally instead of runtime script injection, which is more standard and avoids depending on jsdelivr's uptime; (3) the demo image URL is a temporary/one-off blob storage link — swap for the user's own hosted image. Don't present this as equivalent-risk to the other stash components without calling this out.

```tsx
"use client"
import { useEffect, useRef } from "react"

export function LiquidEffectAnimation() {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const appRef = useRef<any>(null)

  useEffect(() => {
    if (!canvasRef.current) return
    // Load the script dynamically
    const script = document.createElement("script")
    script.type = "module"
    script.textContent = `
      import LiquidBackground from 'https://cdn.jsdelivr.net/npm/threejs-components@0.0.22/build/backgrounds/liquid1.min.js';
      
      const canvas = document.getElementById('liquid-canvas');
      if (canvas) {
        const app = LiquidBackground(canvas);
        app.loadImage('https://hebbkx1anhila5yf.public.blob.vercel-storage.com/enhanced_8bfe61b0-d431-433a-8acb-49d508bf88b4-image-vWzKFKS7vQy7s8wfQYzEpaoiYaVMkr.png');
        app.liquidPlane.material.metalness = 0.75;
        app.liquidPlane.material.roughness = 0.25;
        app.liquidPlane.uniforms.displacementScale.value = 5;
        app.setRain(false);
        window.__liquidApp = app;
      }
    `
    document.body.appendChild(script)
    return () => {
      if (window.__liquidApp && window.__liquidApp.dispose) {
        window.__liquidApp.dispose()
      }
      document.body.removeChild(script)
    }
  }, [])

  return (
    <div
      className="fixed inset-0 m-0 w-full h-full touch-none overflow-hidden"
      style={{ fontFamily: '"Montserrat", serif' }}
    >
      <canvas ref={canvasRef} id="liquid-canvas" className="fixed inset-0 w-full h-full" />
    </div>
  )
}

declare global {
  interface Window {
    __liquidApp?: any
  }
}
```

**Demo usage:**
```tsx
import { LiquidEffectAnimation } from "@/components/ui/liquid-effect-animation";

export default function DemoOne() {
  return <LiquidEffectAnimation />;
}
```

---

## Onboarding Dialog
- **Category**: dialog / modal / onboarding carousel — app UI, not marketing
- **Stack**: React, TypeScript, Tailwind (shadcn-style tokens: `bg-background`, `border-border`, `text-foreground`, `bg-primary`, etc.), `motion/react` (Framer Motion's new package name) for fades/dot animation, `embla-carousel-react` for the slide mechanics
- **Deps**: `motion` (imported as `motion/react`), `embla-carousel-react` — both need installing if not already present. Includes its own tiny local `cn()` helper instead of importing shadcn's `@/lib/utils` — fine to keep or swap for the project's existing `cn()` if one exists (avoid having two `cn()` implementations in the same project).
- **Summary**: A full-screen modal onboarding flow — image carousel (Embla) + animated progress dots + crossfading title/description text (each slide's text stacked in a CSS grid so they crossfade in place rather than causing layout jump) + Back/Skip/Next-or-"Get Started" footer controls. Ships with 4 demo slides using inline-generated gradient placeholder SVGs (via `createPlaceholderImage`) instead of real images — swap `slides` array content and drop the placeholder generator for a real project.
- **Notes**: The placeholder SVGs are a nice trick for prototyping without real assets but shouldn't ship as-is — replace `image: createPlaceholderImage(...)` with real screenshots/illustrations (or pull from Unsplash/Pexels/Pixabay per `references/libraries.md` if genuinely generic imagery is fine). The dialog has no close button in the header and no backdrop-click-to-dismiss — dismissal is only via Skip/Get Started; add an explicit close (X) and/or backdrop click handler if that behavior is expected. `defaultOpen` prop controls initial visibility, and there's a "Restart Onboarding" trigger button rendered when closed — useful for a settings page re-trigger, but likely wants removing/repositioning for a real first-run flow (e.g. open based on a "has the user completed onboarding" flag instead of a hardcoded default).

```tsx
import * as React from "react"
import { motion, AnimatePresence } from "motion/react"
import useEmblaCarousel from "embla-carousel-react"

function cn(...classes: (string | boolean | undefined | null)[]) {
  return classes.filter(Boolean).join(" ")
}

type PlaceholderImageOptions = {
  title: string
  startColor: string
  endColor: string
  accentColor: string
}

const createPlaceholderImage = ({ title, startColor, endColor, accentColor }: PlaceholderImageOptions) => {
  const svg = `
<svg xmlns="http://www.w3.org/2000/svg" width="1200" height="720" viewBox="0 0 1200 720" fill="none">
  <defs>
    <linearGradient id="bg" x1="0" y1="0" x2="1200" y2="720" gradientUnits="userSpaceOnUse">
      <stop stop-color="${startColor}" />
      <stop offset="1" stop-color="${endColor}" />
    </linearGradient>
  </defs>
  <rect width="1200" height="720" rx="40" fill="url(#bg)" />
  <rect x="96" y="96" width="1008" height="528" rx="30" fill="${accentColor}" fill-opacity="0.18" />
  <circle cx="270" cy="220" r="54" fill="${accentColor}" fill-opacity="0.7" />
  <rect x="352" y="184" width="552" height="72" rx="20" fill="${accentColor}" fill-opacity="0.68" />
  <rect x="196" y="350" width="404" height="36" rx="18" fill="${accentColor}" fill-opacity="0.62" />
  <rect x="196" y="410" width="620" height="30" rx="15" fill="${accentColor}" fill-opacity="0.56" />
  <rect x="196" y="456" width="720" height="30" rx="15" fill="${accentColor}" fill-opacity="0.46" />
  <text x="196" y="565" fill="${accentColor}" font-family="Arial, Helvetica, sans-serif" font-size="46" font-weight="700">${title}</text>
</svg>`
  return `data:image/svg+xml;utf8,${encodeURIComponent(svg)}`
}

const slides = [
  {
    id: "welcome",
    alt: "Dashboard preview",
    title: "Welcome to your new workspace",
    description: "Track tasks, documents, and progress from one focused dashboard built for daily execution.",
    image: createPlaceholderImage({ accentColor: "#0B1E47", endColor: "#CDE2FF", startColor: "#EAF2FF", title: "Welcome Dashboard" }),
  },
  {
    id: "automations",
    alt: "Automation workflow preview",
    title: "Automate repetitive work",
    description: "Use smart flows to remove manual busywork and keep your team aligned without extra status meetings.",
    image: createPlaceholderImage({ accentColor: "#0A3D30", endColor: "#CAF6E8", startColor: "#E8FFF7", title: "Automations" }),
  },
  {
    id: "collaboration",
    alt: "Collaboration preview",
    title: "Collaborate in context",
    description: "Share feedback directly where decisions happen so updates stay clear, timely, and easy to follow.",
    image: createPlaceholderImage({ accentColor: "#4A2B00", endColor: "#FFE8C2", startColor: "#FFF6E8", title: "Team Collaboration" }),
  },
  {
    id: "insights",
    alt: "Insights reporting preview",
    title: "Measure outcomes",
    description: "Turn activity into insights with reporting views that highlight what is improving and what needs attention.",
    image: createPlaceholderImage({ accentColor: "#2D1457", endColor: "#E1D4FF", startColor: "#F2ECFF", title: "Insights Reporting" }),
  },
] as const

export function OnboardingDialog({ defaultOpen = true }: { defaultOpen?: boolean }) {
  const [open, setOpen] = React.useState(defaultOpen)
  const [emblaRef, emblaApi] = useEmblaCarousel({ loop: false })
  const [activeIndex, setActiveIndex] = React.useState(0)

  React.useEffect(() => {
    if (!emblaApi) return
    const onSelect = () => setActiveIndex(emblaApi.selectedScrollSnap())
    onSelect()
    emblaApi.on("select", onSelect)
    return () => { emblaApi.off("select", onSelect) }
  }, [emblaApi])

  const isFirstSlide = activeIndex === 0
  const isLastSlide = activeIndex === slides.length - 1
  const currentSlide = slides[activeIndex] ?? slides[0]

  const handleNext = () => {
    if (isLastSlide) { setOpen(false); return }
    emblaApi?.scrollNext()
  }

  const handlePrevious = () => emblaApi?.scrollPrev()

  if (!open) {
    return (
      <button
        onClick={() => { setOpen(true); setTimeout(() => emblaApi?.scrollTo(0), 50) }}
        className="px-4 py-2 rounded-md bg-primary text-primary-foreground text-sm font-medium hover:bg-primary/90"
      >
        Restart Onboarding
      </button>
    )
  }

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      {/* Backdrop */}
      <div className="absolute inset-0 bg-black/60" />

      {/* Dialog */}
      <div className="relative w-full max-w-lg mx-4 rounded-xl bg-background border border-border shadow-2xl overflow-hidden animate-in fade-in zoom-in-95">
        <div className="p-3 sm:p-4">
          {/* Carousel */}
          <div ref={emblaRef} className="overflow-hidden rounded-lg">
            <div className="flex">
              {slides.map((slide) => (
                <div key={slide.id} className="flex-[0_0_100%] min-w-0">
                  <div className="p-1">
                    <img
                      src={slide.image}
                      alt={slide.alt}
                      className="aspect-video w-full rounded-lg object-cover"
                    />
                  </div>
                </div>
              ))}
            </div>
          </div>

          {/* Dots */}
          <div className="flex items-center justify-center gap-2 mt-3">
            {slides.map((slide, index) => (
              <motion.div
                key={slide.id}
                animate={{
                  opacity: index === activeIndex ? 1 : 0.5,
                  width: index === activeIndex ? 24 : 16,
                }}
                initial={false}
                transition={{ duration: 0.22, ease: "easeOut" }}
              >
                <button
                  onClick={() => emblaApi?.scrollTo(index)}
                  aria-label={`Go to ${slide.title}`}
                  className={cn(
                    "h-2 w-full rounded-full transition-colors cursor-pointer",
                    index === activeIndex ? "bg-foreground" : "bg-border hover:bg-muted-foreground"
                  )}
                />
              </motion.div>
            ))}
          </div>

          {/* Title + Description — grid fade */}
          <div className="grid mt-4 px-1">
            {slides.map((slide) => (
              <motion.div
                key={slide.id}
                animate={{ opacity: currentSlide.id === slide.id ? 1 : 0 }}
                initial={false}
                className="col-start-1 row-start-1"
                style={{ pointerEvents: currentSlide.id === slide.id ? "auto" : "none" }}
                transition={{ duration: 0.24, ease: "easeOut" }}
              >
                <h2 className="text-lg font-semibold text-foreground">{slide.title}</h2>
                <p className="text-sm text-muted-foreground mt-2">{slide.description}</p>
              </motion.div>
            ))}
          </div>

          {/* Footer */}
          <div className="flex items-center justify-between mt-6 px-1 pb-1">
            <div>
              {!isFirstSlide && (
                <button
                  onClick={handlePrevious}
                  className="px-3 py-1.5 rounded-md text-sm font-medium text-muted-foreground hover:bg-accent hover:text-foreground transition-colors cursor-pointer"
                >
                  Back
                </button>
              )}
            </div>
            <div className="flex items-center gap-2">
              <button
                onClick={() => setOpen(false)}
                className="px-3 py-1.5 rounded-md text-sm font-medium text-muted-foreground hover:bg-accent hover:text-foreground transition-colors cursor-pointer"
              >
                Skip
              </button>
              <button
                onClick={handleNext}
                className="px-4 py-1.5 rounded-md bg-primary text-primary-foreground text-sm font-medium hover:bg-primary/90 transition-colors cursor-pointer"
              >
                {isLastSlide ? "Get Started" : "Next"}
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  )
}
```

---

## Testimonials V2 (Vertical Scrolling Columns)
- **Category**: marketing section / social proof — self-contained alternative to the earlier "Testimonials with Marquee" entry
- **Stack**: React, TypeScript, Tailwind (with dark mode via `dark:` classes), `framer-motion` (imported as `"framer-motion"`, not `motion/react` like the Onboarding Dialog — check which convention the project already uses before mixing them), `lucide-react` for the Sun/Moon icons
- **Deps**: `framer-motion`, `lucide-react`
- **Summary**: Testimonials laid out in 3 vertical columns that continuously auto-scroll upward (each column's list is duplicated once and translated -50% in an infinite linear loop, so it reads as a seamless endless scroll), each column running at a different `duration` so they drift out of sync for visual variety. Cards have spring-based hover/focus lift + shadow. Whole section fades/rotates in on scroll via `whileInView`. Section edges are softened top/bottom with a `mask-image` gradient so cards fade out at the boundaries instead of hard-cutting. Includes its own dark mode toggle button (`isDark` state toggling a `dark` class on `<html>`) in the demo `App` wrapper — that's demo scaffolding, not part of the reusable section itself.
- **Notes**: This is fully self-contained (data + card markup all inline) — unlike the earlier "Testimonials with Marquee" entry in this file, it does **not** depend on a separate `TestimonialCard` component, so prefer this one when a working testimonials section is needed without extra pieces. Only 3 columns are populated (9 testimonials, sliced 3/3/3); second and third columns are hidden below `md`/`lg` breakpoints respectively, so mobile only shows column 1 — fine as-is, but mention it if the user expects all testimonials visible on mobile. The dark-mode toggle button and `document.documentElement.classList` handling in the `App` export are demo/testbed code — strip those out and just export `TestimonialsSection` (and `TestimonialsColumn` if reused) into the project; don't ship a floating sun/moon toggle unless the user actually wants a real dark mode switch.

```tsx
import React, { useState, useEffect } from 'react';
import { motion } from "framer-motion";
import { Sun, Moon } from 'lucide-react';

interface Testimonial {
  text: string;
  image: string;
  name: string;
  role: string;
}

const testimonials: Testimonial[] = [
  {
    text: "This ERP revolutionized our operations, streamlining finance and inventory. The cloud-based platform keeps us productive, even remotely.",
    image: "https://images.unsplash.com/photo-1494790108377-be9c29b29330?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Briana Patton",
    role: "Operations Manager",
  },
  {
    text: "Implementing this ERP was smooth and quick. The customizable, user-friendly interface made team training effortless.",
    image: "https://images.unsplash.com/photo-1472099645785-5658abf4ff4e?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Bilal Ahmed",
    role: "IT Manager",
  },
  {
    text: "The support team is exceptional, guiding us through setup and providing ongoing assistance, ensuring our satisfaction.",
    image: "https://images.unsplash.com/photo-1438761681033-6461ffad8d80?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Saman Malik",
    role: "Customer Support Lead",
  },
  {
    text: "This ERP's seamless integration enhanced our business operations and efficiency. Highly recommend for its intuitive interface.",
    image: "https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Omar Raza",
    role: "CEO",
  },
  {
    text: "Its robust features and quick support have transformed our workflow, making us significantly more efficient.",
    image: "https://images.unsplash.com/photo-1534528741775-53994a69daeb?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Zainab Hussain",
    role: "Project Manager",
  },
  {
    text: "The smooth implementation exceeded expectations. It streamlined processes, improving overall business performance.",
    image: "https://images.unsplash.com/photo-1517841905240-472988babdf9?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Aliza Khan",
    role: "Business Analyst",
  },
  {
    text: "Our business functions improved with a user-friendly design and positive customer feedback.",
    image: "https://images.unsplash.com/photo-1500648767791-00dcc994a43e?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Farhan Siddiqui",
    role: "Marketing Director",
  },
  {
    text: "They delivered a solution that exceeded expectations, understanding our needs and enhancing our operations.",
    image: "https://images.unsplash.com/photo-1544005313-94ddf0286df2?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Sana Sheikh",
    role: "Sales Manager",
  },
  {
    text: "Using this ERP, our online presence and conversions significantly improved, boosting business performance.",
    image: "https://images.unsplash.com/photo-1506794778202-cad84cf45f1d?auto=format&fit=crop&q=80&w=150&h=150",
    name: "Hassan Ali",
    role: "E-commerce Manager",
  },
];

const firstColumn = testimonials.slice(0, 3);
const secondColumn = testimonials.slice(3, 6);
const thirdColumn = testimonials.slice(6, 9);

const TestimonialsColumn = (props: {
  className?: string;
  testimonials: Testimonial[];
  duration?: number;
}) => {
  return (
    <div className={props.className}>
      <motion.ul
        animate={{
          translateY: "-50%",
        }}
        transition={{
          duration: props.duration || 10,
          repeat: Infinity,
          ease: "linear",
          repeatType: "loop",
        }}
        className="flex flex-col gap-6 pb-6 bg-transparent transition-colors duration-300 list-none m-0 p-0"
      >
        {[
          ...new Array(2).fill(0).map((_, index) => (
            <React.Fragment key={index}>
              {props.testimonials.map(({ text, image, name, role }, i) => (
                <motion.li 
                  key={`${index}-${i}`}
                  aria-hidden={index === 1 ? "true" : "false"}
                  tabIndex={index === 1 ? -1 : 0}
                  whileHover={{ 
                    scale: 1.03,
                    y: -8,
                    boxShadow: "0 25px 50px -12px rgba(0, 0, 0, 0.12), 0 10px 10px -5px rgba(0, 0, 0, 0.04), 0 0 0 1px rgba(0, 0, 0, 0.05)",
                    transition: { type: "spring", stiffness: 400, damping: 17 }
                  }}
                  whileFocus={{ 
                    scale: 1.03,
                    y: -8,
                    boxShadow: "0 25px 50px -12px rgba(0, 0, 0, 0.12), 0 10px 10px -5px rgba(0, 0, 0, 0.04), 0 0 0 1px rgba(0, 0, 0, 0.05)",
                    transition: { type: "spring", stiffness: 400, damping: 17 }
                  }}
                  className="p-10 rounded-3xl border border-neutral-200 dark:border-neutral-800 shadow-lg shadow-black/5 max-w-xs w-full bg-white dark:bg-neutral-900 transition-all duration-300 cursor-default select-none group focus:outline-none focus:ring-2 focus:ring-primary/30" 
                >
                  <blockquote className="m-0 p-0">
                    <p className="text-neutral-600 dark:text-neutral-400 leading-relaxed font-normal m-0 transition-colors duration-300">
                      {text}
                    </p>
                    <footer className="flex items-center gap-3 mt-6">
                      <img
                        width={40}
                        height={40}
                        src={image}
                        alt={`Avatar of ${name}`}
                        className="h-10 w-10 rounded-full object-cover ring-2 ring-neutral-100 dark:ring-neutral-800 group-hover:ring-primary/30 transition-all duration-300 ease-in-out"
                      />
                      <div className="flex flex-col">
                        <cite className="font-semibold not-italic tracking-tight leading-5 text-neutral-900 dark:text-white transition-colors duration-300">
                          {name}
                        </cite>
                        <span className="text-sm leading-5 tracking-tight text-neutral-500 dark:text-neutral-500 mt-0.5 transition-colors duration-300">
                          {role}
                        </span>
                      </div>
                    </footer>
                  </blockquote>
                </motion.li>
              ))}
            </React.Fragment>
          )),
        ]}
      </motion.ul>
    </div>
  );
};

const TestimonialsSection = () => {
  return (
    <section 
      aria-labelledby="testimonials-heading"
      className="bg-transparent py-24 relative overflow-hidden"
    >
      <motion.div 
        initial={{ opacity: 0, y: 50, rotate: -2 }}
        whileInView={{ opacity: 1, y: 0, rotate: 0 }}
        viewport={{ once: true, amount: 0.15 }}
        transition={{ 
          duration: 1.2, 
          ease: [0.16, 1, 0.3, 1],
          opacity: { duration: 0.8 }
        }}
        className="container px-4 z-10 mx-auto"
      >
        <div className="flex flex-col items-center justify-center max-w-[540px] mx-auto mb-16">
          <div className="flex justify-center">
            <div className="border border-neutral-300 dark:border-neutral-700 py-1 px-4 rounded-full text-xs font-semibold tracking-wide uppercase text-neutral-600 dark:text-neutral-400 bg-neutral-100/50 dark:bg-neutral-800/50 transition-colors">
              Testimonials
            </div>
          </div>

          <h2 id="testimonials-heading" className="text-4xl md:text-5xl font-extrabold tracking-tight mt-6 text-center text-neutral-900 dark:text-white transition-colors">
            What our users say
          </h2>
          <p className="text-center mt-5 text-neutral-500 dark:text-neutral-400 text-lg leading-relaxed max-w-sm transition-colors">
            Discover how thousands of teams streamline their operations with our platform.
          </p>
        </div>

        <div 
          className="flex justify-center gap-6 mt-10 [mask-image:linear-gradient(to_bottom,transparent,black_10%,black_90%,transparent)] max-h-[740px] overflow-hidden"
          role="region"
          aria-label="Scrolling Testimonials"
        >
          <TestimonialsColumn testimonials={firstColumn} duration={15} />
          <TestimonialsColumn testimonials={secondColumn} className="hidden md:block" duration={19} />
          <TestimonialsColumn testimonials={thirdColumn} className="hidden lg:block" duration={17} />
        </div>
      </motion.div>
    </section>
  );
};

// Demo wrapper — dark mode toggle here is testbed scaffolding, not part of the reusable section
export default function App() {
  const [isDark, setIsDark] = useState(false);

  useEffect(() => {
    if (isDark) {
      document.documentElement.classList.add('dark');
    } else {
      document.documentElement.classList.remove('dark');
    }
  }, [isDark]);

  return (
    <div className="w-screen min-h-screen bg-white dark:bg-neutral-950 transition-colors duration-300 flex flex-col justify-center relative selection:bg-primary selection:text-white">
      <button 
        onClick={() => setIsDark(!isDark)}
        className="fixed top-6 right-6 z-50 p-3 rounded-full bg-white dark:bg-neutral-900 text-neutral-800 dark:text-neutral-100 border border-neutral-200 dark:border-neutral-800 shadow-xl hover:scale-110 transition-all active:scale-95 focus:outline-none focus:ring-2 focus:ring-primary/50"
        aria-label="Toggle Dark Mode"
      >
        {isDark ? <Sun size={20} /> : <Moon size={20} />}
      </button>

      <TestimonialsSection />
    </div>
  );
}
```

---

## Modern Animated Sign In
- **Category**: auth form / login-signup — large multi-piece bundle, not a single component
- **Stack**: React, TypeScript, Next.js (`next/image`), Tailwind (shadcn `cn()` util), `motion/react` (Framer Motion's new package name — note this is the "motion/react" convention, same as the Onboarding Dialog, but the Testimonials V2 entry above uses the older `"framer-motion"` import — pick one convention per project, don't mix), `lucide-react` (Eye/EyeOff icons)
- **Deps**: `motion`, `lucide-react`, plus `next/image` — if the project isn't Next.js, swap `<Image>` for a plain `<img>` (drop `width`/`height` props or convert as needed)
- **Summary**: A whole auth-page toolkit bundled into one file, exporting several independently reusable pieces: `Input` (spotlight-on-focus glow input), `BoxReveal` (scroll-triggered reveal-through-a-sliding-box text animation), `Ripple` (concentric expanding-ring background decoration), `OrbitingCircles`/`TechOrbitDisplay` (icons orbiting a central label — a decorative side-panel piece, e.g. for showing tech-stack logos next to a login form), `AnimatedForm` (the actual sign-in/sign-up form: header, optional Google login button, fields with per-field `BoxReveal` entrance animation, password visibility toggle, inline validation errors), `AuthTabs` (thin wrapper that centers `AnimatedForm` in a full-height column — named "Tabs" but doesn't actually implement tab-switching between login/signup itself), and `Label`/`BottomGradient` as small helpers.
- **Notes**: This is a component *library within a component* — pull only the pieces actually needed (e.g. just `AnimatedForm` + `Input` + `BoxReveal` + `Label` + `BottomGradient` for a plain login form) rather than importing the whole file if `OrbitingCircles`/`TechOrbitDisplay`/`Ripple` aren't wanted. `AuthTabs`'s name is misleading — despite the name there's no tab UI or login/signup toggle logic inside it; if the user wants an actual tabbed login/signup switcher, that needs to be built on top of this, not assumed to already exist. Form validation is basic (required, email regex, 6-char password minimum) and runs only in `AnimatedForm`'s own `validateForm` — extend it if stronger rules are needed. Google login button just `console.log`s on click — needs real OAuth wiring. Uses CSS variable `var(--skeleton)` for `boxColor` in every `BoxReveal` call — make sure that token is defined in the project's theme, or pass a literal color instead. The dynamic Tailwind class `grid-cols-${fieldPerRow}` **won't work with Tailwind's JIT compiler** unless that exact class string appears literally somewhere for Tailwind to detect at build time — replace with an explicit lookup map (e.g. `{1: 'grid-cols-1', 2: 'grid-cols-2'}[fieldPerRow]`) rather than string interpolation.

```tsx
'use client';
import {
  memo,
  ReactNode,
  useState,
  ChangeEvent,
  FormEvent,
  useEffect,
  useRef,
  forwardRef,
} from 'react';
import Image from 'next/image';
import {
  motion,
  useAnimation,
  useInView,
  useMotionTemplate,
  useMotionValue,
} from 'motion/react';
import { Eye, EyeOff } from 'lucide-react';
import { cn } from '@/lib/utils';

// ==================== Input Component ====================

const Input = memo(
  forwardRef(function Input(
    { className, type, ...props }: React.InputHTMLAttributes<HTMLInputElement>,
    ref: React.ForwardedRef<HTMLInputElement>
  ) {
    const radius = 100;
    const [visible, setVisible] = useState(false);

    const mouseX = useMotionValue(0);
    const mouseY = useMotionValue(0);

    function handleMouseMove({
      currentTarget,
      clientX,
      clientY,
    }: React.MouseEvent<HTMLDivElement>) {
      const { left, top } = currentTarget.getBoundingClientRect();
      mouseX.set(clientX - left);
      mouseY.set(clientY - top);
    }

    return (
      <motion.div
        style={{
          background: useMotionTemplate`
        radial-gradient(
          ${visible ? radius + 'px' : '0px'} circle at ${mouseX}px ${mouseY}px,
          #3b82f6,
          transparent 80%
        )
      `,
        }}
        onMouseMove={handleMouseMove}
        onMouseEnter={() => setVisible(true)}
        onMouseLeave={() => setVisible(false)}
        className='group/input rounded-lg p-[2px] transition duration-300'
      >
        <input
          type={type}
          className={cn(
            `shadow-input dark:placeholder-text-neutral-600 flex h-10 w-full rounded-md border-none bg-gray-50 px-3 py-2 text-sm text-black transition duration-400 group-hover/input:shadow-none file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-neutral-400 focus-visible:ring-[2px] focus-visible:ring-neutral-400 focus-visible:outline-none disabled:cursor-not-allowed disabled:opacity-50 dark:bg-zinc-800 dark:text-white dark:shadow-[0px_0px_1px_1px_#404040] dark:focus-visible:ring-neutral-600`,
            className
          )}
          ref={ref}
          {...props}
        />
      </motion.div>
    );
  })
);

Input.displayName = 'Input';

// ==================== BoxReveal Component ====================

type BoxRevealProps = {
  children: ReactNode;
  width?: string;
  boxColor?: string;
  duration?: number;
  overflow?: string;
  position?: string;
  className?: string;
};

const BoxReveal = memo(function BoxReveal({
  children,
  width = 'fit-content',
  boxColor,
  duration,
  overflow = 'hidden',
  position = 'relative',
  className,
}: BoxRevealProps) {
  const mainControls = useAnimation();
  const slideControls = useAnimation();
  const ref = useRef(null);
  const isInView = useInView(ref, { once: true });

  useEffect(() => {
    if (isInView) {
      slideControls.start('visible');
      mainControls.start('visible');
    } else {
      slideControls.start('hidden');
      mainControls.start('hidden');
    }
  }, [isInView, mainControls, slideControls]);

  return (
    <section
      ref={ref}
      style={{
        position: position as
          | 'relative'
          | 'absolute'
          | 'fixed'
          | 'sticky'
          | 'static',
        width,
        overflow,
      }}
      className={className}
    >
      <motion.div
        variants={{
          hidden: { opacity: 0, y: 75 },
          visible: { opacity: 1, y: 0 },
        }}
        initial='hidden'
        animate={mainControls}
        transition={{ duration: duration ?? 0.5, delay: 0.25 }}
      >
        {children}
      </motion.div>
      <motion.div
        variants={{ hidden: { left: 0 }, visible: { left: '100%' } }}
        initial='hidden'
        animate={slideControls}
        transition={{ duration: duration ?? 0.5, ease: 'easeIn' }}
        style={{
          position: 'absolute',
          top: 4,
          bottom: 4,
          left: 0,
          right: 0,
          zIndex: 20,
          background: boxColor ?? '#5046e6',
          borderRadius: 4,
        }}
      />
    </section>
  );
});

// ==================== Ripple Component ====================

type RippleProps = {
  mainCircleSize?: number;
  mainCircleOpacity?: number;
  numCircles?: number;
  className?: string;
};

const Ripple = memo(function Ripple({
  mainCircleSize = 210,
  mainCircleOpacity = 0.24,
  numCircles = 11,
  className = '',
}: RippleProps) {
  return (
    <section
      className={`max-w-[50%] absolute inset-0 flex items-center justify-center
        dark:bg-white/5 bg-neutral-50
        [mask-image:linear-gradient(to_bottom,black,transparent)]
        dark:[mask-image:linear-gradient(to_bottom,white,transparent)] ${className}`}
    >
      {Array.from({ length: numCircles }, (_, i) => {
        const size = mainCircleSize + i * 70;
        const opacity = mainCircleOpacity - i * 0.03;
        const animationDelay = `${i * 0.06}s`;
        const borderStyle = i === numCircles - 1 ? 'dashed' : 'solid';
        const borderOpacity = 5 + i * 5;

        return (
          <span
            key={i}
            className='absolute animate-ripple rounded-full bg-foreground/15 border'
            style={{
              width: `${size}px`,
              height: `${size}px`,
              opacity: opacity,
              animationDelay: animationDelay,
              borderStyle: borderStyle,
              borderWidth: '1px',
              borderColor: `var(--foreground) dark:var(--background) / ${
                borderOpacity / 100
              })`,
              top: '50%',
              left: '50%',
              transform: 'translate(-50%, -50%)',
            }}
          />
        );
      })}
    </section>
  );
});

// ==================== OrbitingCircles Component ====================

type OrbitingCirclesProps = {
  className?: string;
  children: ReactNode;
  reverse?: boolean;
  duration?: number;
  delay?: number;
  radius?: number;
  path?: boolean;
};

const OrbitingCircles = memo(function OrbitingCircles({
  className,
  children,
  reverse = false,
  duration = 20,
  delay = 10,
  radius = 50,
  path = true,
}: OrbitingCirclesProps) {
  return (
    <>
      {path && (
        <svg
          xmlns='http://www.w3.org/2000/svg'
          version='1.1'
          className='pointer-events-none absolute inset-0 size-full'
        >
          <circle
            className='stroke-black/10 stroke-1 dark:stroke-white/10'
            cx='50%'
            cy='50%'
            r={radius}
            fill='none'
          />
        </svg>
      )}
      <section
        style={
          {
            '--duration': duration,
            '--radius': radius,
            '--delay': -delay,
          } as React.CSSProperties
        }
        className={cn(
          'absolute flex size-full transform-gpu animate-orbit items-center justify-center rounded-full border bg-black/10 [animation-delay:calc(var(--delay)*1000ms)] dark:bg-white/10',
          { '[animation-direction:reverse]': reverse },
          className
        )}
      >
        {children}
      </section>
    </>
  );
});

// ==================== TechOrbitDisplay Component ====================

type IconConfig = {
  className?: string;
  duration?: number;
  delay?: number;
  radius?: number;
  path?: boolean;
  reverse?: boolean;
  component: () => React.ReactNode;
};

type TechnologyOrbitDisplayProps = {
  iconsArray: IconConfig[];
  text?: string;
};

const TechOrbitDisplay = memo(function TechOrbitDisplay({
  iconsArray,
  text = 'Animated Login',
}: TechnologyOrbitDisplayProps) {
  return (
    <section className='relative flex h-full w-full flex-col items-center justify-center overflow-hidden rounded-lg'>
      <span className='pointer-events-none whitespace-pre-wrap bg-gradient-to-b from-black to-gray-300/80 bg-clip-text text-center text-7xl font-semibold leading-none text-transparent dark:from-white dark:to-slate-900/10'>
        {text}
      </span>

      {iconsArray.map((icon, index) => (
        <OrbitingCircles
          key={index}
          className={icon.className}
          duration={icon.duration}
          delay={icon.delay}
          radius={icon.radius}
          path={icon.path}
          reverse={icon.reverse}
        >
          {icon.component()}
        </OrbitingCircles>
      ))}
    </section>
  );
});

// ==================== AnimatedForm Component ====================

type FieldType = 'text' | 'email' | 'password';

type Field = {
  label: string;
  required?: boolean;
  type: FieldType;
  placeholder?: string;
  onChange: (event: ChangeEvent<HTMLInputElement>) => void;
};

type AnimatedFormProps = {
  header: string;
  subHeader?: string;
  fields: Field[];
  submitButton: string;
  textVariantButton?: string;
  errorField?: string;
  fieldPerRow?: number;
  onSubmit: (event: FormEvent<HTMLFormElement>) => void;
  googleLogin?: string;
  goTo?: (event: React.MouseEvent<HTMLButtonElement>) => void;
};

type Errors = {
  [key: string]: string;
};

const AnimatedForm = memo(function AnimatedForm({
  header,
  subHeader,
  fields,
  submitButton,
  textVariantButton,
  errorField,
  fieldPerRow = 1,
  onSubmit,
  googleLogin,
  goTo,
}: AnimatedFormProps) {
  const [visible, setVisible] = useState<boolean>(false);
  const [errors, setErrors] = useState<Errors>({});

  const toggleVisibility = () => setVisible(!visible);

  const validateForm = (event: FormEvent<HTMLFormElement>) => {
    const currentErrors: Errors = {};
    fields.forEach((field) => {
      const value = (event.target as HTMLFormElement)[field.label]?.value;

      if (field.required && !value) {
        currentErrors[field.label] = `${field.label} is required`;
      }

      if (field.type === 'email' && value && !/\S+@\S+\.\S+/.test(value)) {
        currentErrors[field.label] = 'Invalid email address';
      }

      if (field.type === 'password' && value && value.length < 6) {
        currentErrors[field.label] =
          'Password must be at least 6 characters long';
      }
    });
    return currentErrors;
  };

  const handleSubmit = (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    const formErrors = validateForm(event);

    if (Object.keys(formErrors).length === 0) {
      onSubmit(event);
      console.log('Form submitted');
    } else {
      setErrors(formErrors);
    }
  };

  return (
    <section className='max-md:w-full flex flex-col gap-4 w-96 mx-auto'>
      <BoxReveal boxColor='var(--skeleton)' duration={0.3}>
        <h2 className='font-bold text-3xl text-neutral-800 dark:text-neutral-200'>
          {header}
        </h2>
      </BoxReveal>

      {subHeader && (
        <BoxReveal boxColor='var(--skeleton)' duration={0.3} className='pb-2'>
          <p className='text-neutral-600 text-sm max-w-sm dark:text-neutral-300'>
            {subHeader}
          </p>
        </BoxReveal>
      )}

      {googleLogin && (
        <>
          <BoxReveal
            boxColor='var(--skeleton)'
            duration={0.3}
            overflow='visible'
            width='unset'
          >
            <button
              className='g-button group/btn bg-transparent w-full rounded-md border h-10 font-medium outline-hidden hover:cursor-pointer'
              type='button'
              onClick={() => console.log('Google login clicked')}
            >
              <span className='flex items-center justify-center w-full h-full gap-3'>
                <Image
                  src='https://cdn1.iconfinder.com/data/icons/google-s-logo/150/Google_Icons-09-512.png'
                  width={26}
                  height={26}
                  alt='Google Icon'
                />
                {googleLogin}
              </span>

              <BottomGradient />
            </button>
          </BoxReveal>

          <BoxReveal boxColor='var(--skeleton)' duration={0.3} width='100%'>
            <section className='flex items-center gap-4'>
              <hr className='flex-1 border-1 border-dashed border-neutral-300 dark:border-neutral-700' />
              <p className='text-neutral-700 text-sm dark:text-neutral-300'>
                or
              </p>
              <hr className='flex-1 border-1 border-dashed border-neutral-300 dark:border-neutral-700' />
            </section>
          </BoxReveal>
        </>
      )}

      <form onSubmit={handleSubmit}>
        <section
          className={`grid grid-cols-1 md:grid-cols-${fieldPerRow} mb-4`}
        >
          {fields.map((field) => (
            <section key={field.label} className='flex flex-col gap-2'>
              <BoxReveal boxColor='var(--skeleton)' duration={0.3}>
                <Label htmlFor={field.label}>
                  {field.label} <span className='text-red-500'>*</span>
                </Label>
              </BoxReveal>

              <BoxReveal
                width='100%'
                boxColor='var(--skeleton)'
                duration={0.3}
                className='flex flex-col space-y-2 w-full'
              >
                <section className='relative'>
                  <Input
                    type={
                      field.type === 'password'
                        ? visible
                          ? 'text'
                          : 'password'
                        : field.type
                    }
                    id={field.label}
                    placeholder={field.placeholder}
                    onChange={field.onChange}
                  />

                  {field.type === 'password' && (
                    <button
                      type='button'
                      onClick={toggleVisibility}
                      className='absolute inset-y-0 right-0 pr-3 flex items-center text-sm leading-5'
                    >
                      {visible ? (
                        <Eye className='h-5 w-5' />
                      ) : (
                        <EyeOff className='h-5 w-5' />
                      )}
                    </button>
                  )}
                </section>

                <section className='h-4'>
                  {errors[field.label] && (
                    <p className='text-red-500 text-xs'>
                      {errors[field.label]}
                    </p>
                  )}
                </section>
              </BoxReveal>
            </section>
          ))}
        </section>

        <BoxReveal width='100%' boxColor='var(--skeleton)' duration={0.3}>
          {errorField && (
            <p className='text-red-500 text-sm mb-4'>{errorField}</p>
          )}
        </BoxReveal>

        <BoxReveal
          width='100%'
          boxColor='var(--skeleton)'
          duration={0.3}
          overflow='visible'
        >
          <button
            className='bg-gradient-to-br relative group/btn from-zinc-200 dark:from-zinc-900
            dark:to-zinc-900 to-zinc-200 block dark:bg-zinc-800 w-full text-black
            dark:text-white rounded-md h-10 font-medium shadow-[0px_1px_0px_0px_#ffffff40_inset,0px_-1px_0px_0px_#ffffff40_inset] 
              dark:shadow-[0px_1px_0px_0px_var(--zinc-800)_inset,0px_-1px_0px_0px_var(--zinc-800)_inset] outline-hidden hover:cursor-pointer'
            type='submit'
          >
            {submitButton} &rarr;
            <BottomGradient />
          </button>
        </BoxReveal>

        {textVariantButton && goTo && (
          <BoxReveal boxColor='var(--skeleton)' duration={0.3}>
            <section className='mt-4 text-center hover:cursor-pointer'>
              <button
                className='text-sm text-blue-500 hover:cursor-pointer outline-hidden'
                onClick={goTo}
              >
                {textVariantButton}
              </button>
            </section>
          </BoxReveal>
        )}
      </form>
    </section>
  );
});

const BottomGradient = () => {
  return (
    <>
      <span className='group-hover/btn:opacity-100 block transition duration-500 opacity-0 absolute h-px w-full -bottom-px inset-x-0 bg-gradient-to-r from-transparent via-cyan-500 to-transparent' />
      <span className='group-hover/btn:opacity-100 blur-sm block transition duration-500 opacity-0 absolute h-px w-1/2 mx-auto -bottom-px inset-x-10 bg-gradient-to-r from-transparent via-indigo-500 to-transparent' />
    </>
  );
};

// ==================== AuthTabs Component ====================

interface AuthTabsProps {
  formFields: {
    header: string;
    subHeader?: string;
    fields: Array<{
      label: string;
      required?: boolean;
      type: string;
      placeholder: string;
      onChange: (event: React.ChangeEvent<HTMLInputElement>) => void;
    }>;
    submitButton: string;
    textVariantButton?: string;
  };
  goTo: (event: React.MouseEvent<HTMLButtonElement>) => void;
  handleSubmit: (event: React.FormEvent<HTMLFormElement>) => void;
}

const AuthTabs = memo(function AuthTabs({
  formFields,
  goTo,
  handleSubmit,
}: AuthTabsProps) {
  return (
    <div className='flex max-lg:justify-center w-full md:w-auto'>
      <div className='w-full lg:w-1/2 h-[100dvh] flex flex-col justify-center items-center max-lg:px-[10%]'>
        <AnimatedForm
          {...formFields}
          fieldPerRow={1}
          onSubmit={handleSubmit}
          goTo={goTo}
          googleLogin='Login with Google'
        />
      </div>
    </div>
  );
});

// ==================== Label Component ====================

interface LabelProps extends React.LabelHTMLAttributes<HTMLLabelElement> {
  htmlFor?: string;
}

const Label = memo(function Label({ className, ...props }: LabelProps) {
  return (
    <label
      className={cn(
        'text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70',
        className
      )}
      {...props}
    />
  );
});

// ==================== Exports ====================

export {
  Input,
  BoxReveal,
  Ripple,
  OrbitingCircles,
  TechOrbitDisplay,
  AnimatedForm,
  AuthTabs,
  Label,
  BottomGradient,
};
```

---

## Pricing
- **Category**: marketing section / pricing table
- **Stack**: React, TypeScript, Next.js (`next/link`), Tailwind (shadcn `cn()`, `buttonVariants`, `Label`, `Switch` components), `framer-motion` (older `"framer-motion"` import — matches the Testimonials V2 entry's convention, not the `motion/react` one used by Onboarding Dialog / Modern Animated Sign In; check which the project already uses), `lucide-react` (Check, Star icons), `canvas-confetti`, `@number-flow/react`
- **Deps**: `framer-motion`, `lucide-react`, `canvas-confetti`, `@number-flow/react`, plus shadcn's `Label`, `Switch`, `Button` (for `buttonVariants`) components and a `useMediaQuery` hook — several of these need to already exist in the project (standard shadcn setup covers `Label`/`Switch`/`Button`, but `useMediaQuery` is a custom hook not included here and needs writing or fetching separately)
- **Summary**: A 3-tier pricing section with a monthly/annual billing toggle. Flipping the toggle to annual fires a confetti burst (`canvas-confetti`) positioned at the switch's location, and animates the displayed price with `NumberFlow` (smooth digit-rolling transition between the monthly/yearly numbers) rather than an instant swap. On desktop, the "popular" plan is visually lifted (`y: -20`) while the two side plans are subtly rotated/scaled/pushed outward to create a 3D "featured card" carousel effect; this whole motion/rotation treatment is skipped on mobile (`isDesktop` check via `useMediaQuery`). Popular plan gets a ribbon badge (Star icon) in the corner.
- **Notes**: `useMediaQuery` (`@/hooks/use-media-query`) is referenced but not defined in this file — needs a small custom hook (matches a media query string, returns boolean, updates on resize) written or fetched before this compiles. Price display currently does `Number(plan.price)` on a plain string like `"50"` and formats it as USD currency — if non-USD pricing or decimals are needed, adjust the `NumberFlow` `format` prop. The 3D-tilt classes (`-translate-z-[50px] rotate-y-[10deg]`, etc.) rely on Tailwind's 3D transform utilities and a parent with `perspective` set somewhere in scope for the rotation to read as remotely 3D rather than just a flat rotate — verify the project's Tailwind version supports these utilities (fairly recent Tailwind feature) and that there's a `perspective` ancestor, or the tilt may look wrong/flat. Confetti colors reference CSS variables (`hsl(var(--primary))` etc.) assuming a standard shadcn theme setup — adjust if the project uses different token names.

```tsx
"use client";

import { buttonVariants } from "@/components/ui/button";
import { Label } from "@/components/ui/label";
import { Switch } from "@/components/ui/switch";
import { useMediaQuery } from "@/hooks/use-media-query";
import { cn } from "@/lib/utils";
import { motion } from "framer-motion";
import { Check, Star } from "lucide-react";
import Link from "next/link";
import { useState, useRef } from "react";
import confetti from "canvas-confetti";
import NumberFlow from "@number-flow/react";

interface PricingPlan {
  name: string;
  price: string;
  yearlyPrice: string;
  period: string;
  features: string[];
  description: string;
  buttonText: string;
  href: string;
  isPopular: boolean;
}

interface PricingProps {
  plans: PricingPlan[];
  title?: string;
  description?: string;
}

export function Pricing({
  plans,
  title = "Simple, Transparent Pricing",
  description = "Choose the plan that works for you\nAll plans include access to our platform, lead generation tools, and dedicated support.",
}: PricingProps) {
  const [isMonthly, setIsMonthly] = useState(true);
  const isDesktop = useMediaQuery("(min-width: 768px)");
  const switchRef = useRef<HTMLButtonElement>(null);

  const handleToggle = (checked: boolean) => {
    setIsMonthly(!checked);
    if (checked && switchRef.current) {
      const rect = switchRef.current.getBoundingClientRect();
      const x = rect.left + rect.width / 2;
      const y = rect.top + rect.height / 2;

      confetti({
        particleCount: 50,
        spread: 60,
        origin: {
          x: x / window.innerWidth,
          y: y / window.innerHeight,
        },
        colors: [
          "hsl(var(--primary))",
          "hsl(var(--accent))",
          "hsl(var(--secondary))",
          "hsl(var(--muted))",
        ],
        ticks: 200,
        gravity: 1.2,
        decay: 0.94,
        startVelocity: 30,
        shapes: ["circle"],
      });
    }
  };

  return (
    <div className="container py-20">
      <div className="text-center space-y-4 mb-12">
        <h2 className="text-4xl font-bold tracking-tight sm:text-5xl">
          {title}
        </h2>
        <p className="text-muted-foreground text-lg whitespace-pre-line">
          {description}
        </p>
      </div>

      <div className="flex justify-center mb-10">
        <label className="relative inline-flex items-center cursor-pointer">
          <Label>
            <Switch
              ref={switchRef as any}
              checked={!isMonthly}
              onCheckedChange={handleToggle}
              className="relative"
            />
          </Label>
        </label>
        <span className="ml-2 font-semibold">
          Annual billing <span className="text-primary">(Save 20%)</span>
        </span>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 sm:2 gap-4">
        {plans.map((plan, index) => (
          <motion.div
            key={index}
            initial={{ y: 50, opacity: 1 }}
            whileInView={
              isDesktop
                ? {
                    y: plan.isPopular ? -20 : 0,
                    opacity: 1,
                    x: index === 2 ? -30 : index === 0 ? 30 : 0,
                    scale: index === 0 || index === 2 ? 0.94 : 1.0,
                  }
                : {}
            }
            viewport={{ once: true }}
            transition={{
              duration: 1.6,
              type: "spring",
              stiffness: 100,
              damping: 30,
              delay: 0.4,
              opacity: { duration: 0.5 },
            }}
            className={cn(
              `rounded-2xl border-[1px] p-6 bg-background text-center lg:flex lg:flex-col lg:justify-center relative`,
              plan.isPopular ? "border-primary border-2" : "border-border",
              "flex flex-col",
              !plan.isPopular && "mt-5",
              index === 0 || index === 2
                ? "z-0 transform translate-x-0 translate-y-0 -translate-z-[50px] rotate-y-[10deg]"
                : "z-10",
              index === 0 && "origin-right",
              index === 2 && "origin-left"
            )}
          >
            {plan.isPopular && (
              <div className="absolute top-0 right-0 bg-primary py-0.5 px-2 rounded-bl-xl rounded-tr-xl flex items-center">
                <Star className="text-primary-foreground h-4 w-4 fill-current" />
                <span className="text-primary-foreground ml-1 font-sans font-semibold">
                  Popular
                </span>
              </div>
            )}
            <div className="flex-1 flex flex-col">
              <p className="text-base font-semibold text-muted-foreground">
                {plan.name}
              </p>
              <div className="mt-6 flex items-center justify-center gap-x-2">
                <span className="text-5xl font-bold tracking-tight text-foreground">
                  <NumberFlow
                    value={
                      isMonthly ? Number(plan.price) : Number(plan.yearlyPrice)
                    }
                    format={{
                      style: "currency",
                      currency: "USD",
                      minimumFractionDigits: 0,
                      maximumFractionDigits: 0,
                    }}
                    formatter={(value) => `$${value}`}
                    transformTiming={{
                      duration: 500,
                      easing: "ease-out",
                    }}
                    willChange
                    className="font-variant-numeric: tabular-nums"
                  />
                </span>
                {plan.period !== "Next 3 months" && (
                  <span className="text-sm font-semibold leading-6 tracking-wide text-muted-foreground">
                    / {plan.period}
                  </span>
                )}
              </div>

              <p className="text-xs leading-5 text-muted-foreground">
                {isMonthly ? "billed monthly" : "billed annually"}
              </p>

              <ul className="mt-5 gap-2 flex flex-col">
                {plan.features.map((feature, idx) => (
                  <li key={idx} className="flex items-start gap-2">
                    <Check className="h-4 w-4 text-primary mt-1 flex-shrink-0" />
                    <span className="text-left">{feature}</span>
                  </li>
                ))}
              </ul>

              <hr className="w-full my-4" />

              <Link
                href={plan.href}
                className={cn(
                  buttonVariants({
                    variant: "outline",
                  }),
                  "group relative w-full gap-2 overflow-hidden text-lg font-semibold tracking-tighter",
                  "transform-gpu ring-offset-current transition-all duration-300 ease-out hover:ring-2 hover:ring-primary hover:ring-offset-1 hover:bg-primary hover:text-primary-foreground",
                  plan.isPopular
                    ? "bg-primary text-primary-foreground"
                    : "bg-background text-foreground"
                )}
              >
                {plan.buttonText}
              </Link>
              <p className="mt-6 text-xs leading-5 text-muted-foreground">
                {plan.description}
              </p>
            </div>
          </motion.div>
        ))}
      </div>
    </div>
  );
}
```

**Demo usage:**
```tsx
"use client";

import { Pricing } from "@/components/blocks/pricing";

const demoPlans = [
  {
    name: "STARTER",
    price: "50",
    yearlyPrice: "40",
    period: "per month",
    features: [
      "Up to 10 projects",
      "Basic analytics",
      "48-hour support response time",
      "Limited API access",
      "Community support",
    ],
    description: "Perfect for individuals and small projects",
    buttonText: "Start Free Trial",
    href: "/sign-up",
    isPopular: false,
  },
  {
    name: "PROFESSIONAL",
    price: "99",
    yearlyPrice: "79",
    period: "per month",
    features: [
      "Unlimited projects",
      "Advanced analytics",
      "24-hour support response time",
      "Full API access",
      "Priority support",
      "Team collaboration",
      "Custom integrations",
    ],
    description: "Ideal for growing teams and businesses",
    buttonText: "Get Started",
    href: "/sign-up",
    isPopular: true,
  },
  {
    name: "ENTERPRISE",
    price: "299",
    yearlyPrice: "239",
    period: "per month",
    features: [
      "Everything in Professional",
      "Custom solutions",
      "Dedicated account manager",
      "1-hour support response time",
      "SSO Authentication",
      "Advanced security",
      "Custom contracts",
      "SLA agreement",
    ],
    description: "For large organizations with specific needs",
    buttonText: "Contact Sales",
    href: "/contact",
    isPopular: false,
  },
];

function PricingBasic() {
  return (
    <div className="h-[800px] overflow-y-auto rounded-lg">
      <Pricing 
        plans={demoPlans}
        title="Simple, Transparent Pricing"
        description="Choose the plan that works for you\nAll plans include access to our platform, lead generation tools, and dedicated support."
      />
    </div>
  );
}

export { PricingBasic };
```

---

## Prisma Hero
- **Category**: hero section / full-screen video hero — landing page block
- **Stack**: React, TypeScript, Tailwind, `framer-motion` (older `"framer-motion"` import), `lucide-react` (ArrowRight)
- **Deps**: `framer-motion`, `lucide-react` — no other outside packages, but see notes on the missing CSS class below
- **Summary**: A full-viewport hero with an autoplaying looped background video, a film-grain noise overlay (`mix-blend-overlay`), and a top/bottom gradient for text legibility. Includes two reusable text-animation exports — `WordsPullUp` (splits a string into words, each fading/sliding up on scroll-into-view with a staggered delay; supports an optional trailing asterisk superscript) and `WordsPullUpMultiStyle` (same effect but takes styled text segments so different words/phrases can have different classes, e.g. mixing weights or colors within one animated headline). A pill-shaped floating nav sits centered at the top, and the bottom of the hero holds a huge fluid-width headline (`text-[26vw]` down to `text-[19vw]` across breakpoints) using `WordsPullUp`, plus a description paragraph and a CTA button with an animated icon-circle that grows on hover.
- **Notes**: References a `.noise-overlay` CSS class that is **not defined anywhere in this file** — it needs a grain/noise texture (typically a small tiling background-image PNG/SVG or an `background-image: url(data:...)` noise texture) added to the project's global CSS, or the overlay div will render as nothing. The background `<video>` `src` and nav items/copy ("Prisma", "Join the lab", "Our story"/"Collective"/etc.) are all specific to the original demo brand — swap the video URL, headline text, nav items, and CTA copy for the actual project. Color values are hardcoded as literal hex/rgba (`#E1E0CC`, `rgba(225, 224, 204, 0.8)`) rather than theme tokens except for `bg-primary`/`text-primary` — decide whether to convert those hardcoded colors into the project's theme variables for consistency, or keep them as an intentional one-off brand accent. The headline font-size scale is extremely viewport-dependent (`vw` units) — on very wide or very narrow viewports outside typical breakpoints this can get either huge or tiny; sanity-check on the actual target device sizes.

```tsx
import { motion, useInView } from "framer-motion";
import { ArrowRight } from "lucide-react";
import { useRef } from "react";

/* ---------------- WordsPullUp ---------------- */
interface WordsPullUpProps {
  text: string;
  className?: string;
  showAsterisk?: boolean;
  style?: React.CSSProperties;
}

export const WordsPullUp = ({ text, className = "", showAsterisk = false, style }: WordsPullUpProps) => {
  const ref = useRef<HTMLDivElement>(null);
  const isInView = useInView(ref, { once: true });
  const words = text.split(" ");

  return (
    <div ref={ref} className={`inline-flex flex-wrap ${className}`} style={style}>
      {words.map((word, i) => {
        const isLast = i === words.length - 1;
        return (
          <motion.span
            key={i}
            initial={{ y: 20, opacity: 0 }}
            animate={isInView ? { y: 0, opacity: 1 } : {}}
            transition={{ duration: 0.6, delay: i * 0.08, ease: [0.16, 1, 0.3, 1] }}
            className="inline-block relative"
            style={{ marginRight: isLast ? 0 : "0.25em" }}
          >
            {word}
            {showAsterisk && isLast && (
              <span className="absolute top-[0.65em] -right-[0.3em] text-[0.31em]">*</span>
            )}
          </motion.span>
        );
      })}
    </div>
  );
};

/* ---------------- WordsPullUpMultiStyle ---------------- */
interface Segment {
  text: string;
  className?: string;
}

interface WordsPullUpMultiStyleProps {
  segments: Segment[];
  className?: string;
  style?: React.CSSProperties;
}

export const WordsPullUpMultiStyle = ({ segments, className = "", style }: WordsPullUpMultiStyleProps) => {
  const ref = useRef<HTMLDivElement>(null);
  const isInView = useInView(ref, { once: true });

  const words: { word: string; className?: string }[] = [];
  segments.forEach((seg) => {
    seg.text.split(" ").forEach((w) => {
      if (w) words.push({ word: w, className: seg.className });
    });
  });

  return (
    <div ref={ref} className={`inline-flex flex-wrap justify-center ${className}`} style={style}>
      {words.map((w, i) => (
        <motion.span
          key={i}
          initial={{ y: 20, opacity: 0 }}
          animate={isInView ? { y: 0, opacity: 1 } : {}}
          transition={{ duration: 0.6, delay: i * 0.08, ease: [0.16, 1, 0.3, 1] }}
          className={`inline-block ${w.className ?? ""}`}
          style={{ marginRight: "0.25em" }}
        >
          {w.word}
        </motion.span>
      ))}
    </div>
  );
};

/* ---------------- Hero ---------------- */
const navItems = ["Our story", "Collective", "Workshops", "Programs", "Inquiries"];

const PrismaHero = () => {
  return (
    <section className="h-screen w-full">
      <div className="relative h-full w-full overflow-hidden rounded-2xl md:rounded-[2rem]">
        
        {/* Background video */}
        <video
          autoPlay
          loop
          muted
          playsInline
          className="absolute inset-0 h-full w-full object-cover"
          src="https://example.com/hero-video.mp4"
        />

        {/* Noise overlay */}
        <div className="noise-overlay pointer-events-none absolute inset-0 opacity-[0.7] mix-blend-overlay" />

        {/* Gradient overlay */}
        <div className="pointer-events-none absolute inset-0 bg-gradient-to-b from-black/30 via-transparent to-black/60" />

        {/* Navbar */}
        <nav className="absolute left-1/2 top-0 z-20 -translate-x-1/2">
          <div className="flex items-center gap-3 rounded-b-2xl bg-black px-4 py-2 sm:gap-6 md:gap-12 md:rounded-b-3xl md:px-8 lg:gap-14">
            {navItems.map((item) => (
              <a
                key={item}
                href="#"
                className="text-[10px] transition-colors sm:text-xs md:text-sm"
                style={{ color: "rgba(225, 224, 204, 0.8)" }}
                onMouseEnter={(e) => (e.currentTarget.style.color = "#E1E0CC")}
                onMouseLeave={(e) => (e.currentTarget.style.color = "rgba(225, 224, 204, 0.8)")}
              >
                {item}
              </a>
            ))}
          </div>
        </nav>

        {/* Hero content */}
        <div className="absolute bottom-0 left-0 right-0 px-4 pb-2 sm:px-6 md:px-10">
          <div className="grid grid-cols-12 items-end gap-4">
            
            <div className="col-span-12 lg:col-span-8">
              <h1
                className="font-medium leading-[0.85] tracking-[-0.07em] text-[26vw] sm:text-[24vw] md:text-[22vw] lg:text-[20vw] xl:text-[19vw] 2xl:text-[20vw]"
                style={{ color: "#E1E0CC" }}
              >
                <WordsPullUp text="Prisma" showAsterisk />
              </h1>
            </div>

            <div className="col-span-12 flex flex-col gap-5 pb-6 lg:col-span-4 lg:pb-10">
              
              <motion.p
                initial={{ y: 20, opacity: 0 }}
                animate={{ y: 0, opacity: 1 }}
                transition={{ duration: 0.8, delay: 0.5, ease: [0.16, 1, 0.3, 1] }}
                className="text-xs text-primary/70 sm:text-sm md:text-base"
                style={{ lineHeight: 1.2 }}
              >
                Prisma is a worldwide network of visual artists, filmmakers and storytellers bound not by place, status or labels but by passion and hunger to unlock potential through our unique perspectives.
              </motion.p>

              <motion.button
                initial={{ y: 20, opacity: 0 }}
                animate={{ y: 0, opacity: 1 }}
                transition={{ duration: 0.8, delay: 0.7, ease: [0.16, 1, 0.3, 1] }}
                className="group inline-flex items-center gap-2 self-start rounded-full bg-primary py-1 pl-5 pr-1 text-sm font-medium text-black transition-all hover:gap-3 sm:text-base"
              >
                Join the lab
                <span className="flex h-9 w-9 items-center justify-center rounded-full bg-black transition-transform group-hover:scale-110 sm:h-10 sm:w-10">
                  <ArrowRight className="h-4 w-4" style={{ color: "#E1E0CC" }} />
                </span>
              </motion.button>

            </div>
          </div>
        </div>
      </div>
    </section>
  );
};

export { PrismaHero };
```

---

## Floating Icons Hero Section
- **Category**: hero section / "trusted by" logo cloud — landing page block
- **Stack**: React, TypeScript, Tailwind (shadcn `cn()` + `Button`), `framer-motion` (older `"framer-motion"` import, uses `useMotionValue`/`useSpring` for physics-based movement, not just simple transitions)
- **Deps**: `framer-motion`, plus shadcn's `Button` component
- **Summary**: A hero with a centered headline/subtitle/CTA over a field of floating brand-logo icons scattered by percentage-based positions. Each icon has two motion layers: a spring-physics cursor-repulsion effect (icons push away when the mouse gets within 150px, using `Math.atan2`/distance math against a global `mousemove` listener) plus a separate continuous idle float/rotate loop on an inner wrapper, so icons drift on their own even without mouse interaction. Entrance is staggered by index. Component is generic — the actual icon set (which logos, how many, their positions) is entirely supplied via the `icons` prop, so `FloatingIconsHero` itself has no brand dependency.
- **Notes — flag before using**: The **demo** file (not the component itself) hardcodes 16 real, recognizable brand logos as inline SVGs — Google, Apple, Microsoft, Figma, GitHub, Slack, Vercel, Stripe, Discord, X, Spotify, Dropbox, Twitch, Linear, YouTube, Notion. These are trademarked logos; don't ship them as-is on a real project implying those companies are customers/partners/sponsors unless that's actually true and permitted — that's a trademark/brand-misuse issue, not just a style concern. For a real project: either (a) swap in the user's actual customer/partner logos, (b) use a neutral icon set (e.g. Reicon or another icon library from `references/libraries.md`) instead of brand marks, or (c) confirm with the user that they have rights to display these specific logos (e.g. an actual integrations/"works with" page) before reusing them. The base `FloatingIconsHero` component itself is generic and fine to reuse — it's specifically the demo's icon choices that need a decision before shipping. Also note: the mouse-repulsion effect attaches a `window`-level `mousemove` listener per icon (one listener per rendered icon) — fine for ~16 icons, but if scaling up to many more icons, consider consolidating to a single shared listener for performance.

```tsx
import * as React from 'react';
import { motion, useMotionValue, useSpring } from 'framer-motion';
import { cn } from '@/lib/utils';
import { Button } from '@/components/ui/button';

interface IconProps {
  id: number;
  icon: React.FC<React.SVGProps<SVGSVGElement>>;
  className: string;
}

export interface FloatingIconsHeroProps {
  title: string;
  subtitle: string;
  ctaText: string;
  ctaHref: string;
  icons: IconProps[];
}

const Icon = ({
  mouseX,
  mouseY,
  iconData,
  index,
}: {
  mouseX: React.MutableRefObject<number>;
  mouseY: React.MutableRefObject<number>;
  iconData: IconProps;
  index: number;
}) => {
  const ref = React.useRef<HTMLDivElement>(null);

  const x = useMotionValue(0);
  const y = useMotionValue(0);
  const springX = useSpring(x, { stiffness: 300, damping: 20 });
  const springY = useSpring(y, { stiffness: 300, damping: 20 });

  React.useEffect(() => {
    const handleMouseMove = () => {
      if (ref.current) {
        const rect = ref.current.getBoundingClientRect();
        const distance = Math.sqrt(
          Math.pow(mouseX.current - (rect.left + rect.width / 2), 2) +
            Math.pow(mouseY.current - (rect.top + rect.height / 2), 2)
        );

        if (distance < 150) {
          const angle = Math.atan2(
            mouseY.current - (rect.top + rect.height / 2),
            mouseX.current - (rect.left + rect.width / 2)
          );
          const force = (1 - distance / 150) * 50;
          x.set(-Math.cos(angle) * force);
          y.set(-Math.sin(angle) * force);
        } else {
          x.set(0);
          y.set(0);
        }
      }
    };

    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, [x, y, mouseX, mouseY]);

  return (
    <motion.div
      ref={ref}
      key={iconData.id}
      style={{
        x: springX,
        y: springY,
      }}
      initial={{ opacity: 0, scale: 0.5 }}
      animate={{ opacity: 1, scale: 1 }}
      transition={{
        delay: index * 0.08,
        duration: 0.6,
        ease: [0.22, 1, 0.36, 1],
      }}
      className={cn('absolute', iconData.className)}
    >
      <motion.div
        className="flex items-center justify-center w-16 h-16 md:w-20 md:h-20 p-3 rounded-3xl shadow-xl bg-card/80 backdrop-blur-md border border-border/10"
        animate={{
          y: [0, -8, 0, 8, 0],
          x: [0, 6, 0, -6, 0],
          rotate: [0, 5, 0, -5, 0],
        }}
        transition={{
          duration: 5 + Math.random() * 5,
          repeat: Infinity,
          repeatType: 'mirror',
          ease: 'easeInOut',
        }}
      >
        <iconData.icon className="w-8 h-8 md:w-10 md:h-10 text-foreground" />
      </motion.div>
    </motion.div>
  );
};

const FloatingIconsHero = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement> & FloatingIconsHeroProps
>(({ className, title, subtitle, ctaText, ctaHref, icons, ...props }, ref) => {
  const mouseX = React.useRef(0);
  const mouseY = React.useRef(0);

  const handleMouseMove = (event: React.MouseEvent<HTMLDivElement>) => {
    mouseX.current = event.clientX;
    mouseY.current = event.clientY;
  };

  return (
    <section
      ref={ref}
      onMouseMove={handleMouseMove}
      className={cn(
        'relative w-full h-screen min-h-[700px] flex items-center justify-center overflow-hidden bg-background',
        className
      )}
      {...props}
    >
      <div className="absolute inset-0 w-full h-full">
        {icons.map((iconData, index) => (
          <Icon
            key={iconData.id}
            mouseX={mouseX}
            mouseY={mouseY}
            iconData={iconData}
            index={index}
          />
        ))}
      </div>

      <div className="relative z-10 text-center px-4">
        <h1 className="text-5xl md:text-7xl font-bold tracking-tight bg-gradient-to-b from-foreground to-foreground/70 text-transparent bg-clip-text">
          {title}
        </h1>
        <p className="mt-6 max-w-xl mx-auto text-lg text-muted-foreground">
          {subtitle}
        </p>
        <div className="mt-10">
          <Button asChild size="lg" className="px-8 py-6 text-base font-semibold">
            <a href={ctaHref}>{ctaText}</a>
          </Button>
        </div>
      </div>
    </section>
  );
});

FloatingIconsHero.displayName = 'FloatingIconsHero';

export { FloatingIconsHero };
```

**Demo usage** (⚠️ demo icon set uses real brand logos — see notes above before reusing as-is):
```tsx
import * as React from 'react';
import {
  FloatingIconsHero,
  type FloatingIconsHeroProps,
} from '@/components/ui/floating-icons-hero-section';

// Demo supplies 16 inline brand-logo SVGs here (Google, Apple, Microsoft, Figma,
// GitHub, Slack, Vercel, Stripe, Discord, X, Spotify, Dropbox, Twitch, Linear,
// YouTube, Notion) each positioned via a `className` with percentage-based
// top/left/right/bottom offsets. Omitted here — swap for neutral icons or the
// user's actual partner logos when reusing.

const demoIcons: FloatingIconsHeroProps['icons'] = [
  { id: 1, icon: /* IconGoogle */ undefined as any, className: 'top-[10%] left-[10%]' },
  // ...remaining icons follow the same { id, icon, className } shape
];

export default function FloatingIconsHeroDemo() {
  return (
    <FloatingIconsHero
      title="A World of Innovation"
      subtitle="Explore a universe of possibilities with our platform, connecting you to the tools and technologies that shape the future."
      ctaText="Join the Revolution"
      ctaHref="#"
      icons={demoIcons}
    />
  );
}
```

---

## Globe Study (Interactive Text Globe)
- **Category**: interactive canvas art / decorative visualization — unusual, high-effort piece
- **Stack**: React wrapper around a fully self-contained vanilla-JS/HTML/Canvas2D "study" — NOT a normal React component internally. The entire visualization (~400 lines of plain JS driving a `<canvas>`) is embedded as a giant HTML string and rendered inside a sandboxed `<iframe srcDoc={...}>`. Only the outer wrapper (`mode`/`scale`/`opacity`/`hue`/`saturation`/`brightness` props, light/dark theming, CSS filter effects) is real React/TypeScript.
- **Deps**: none beyond React — no canvas/animation libraries, it's hand-rolled Canvas2D math
- **Source/license note**: The embedded HTML/JS is explicitly commented in the source as adapted from **MengTo/threeui** (MIT licensed), specifically its "text-on-a-path" shader studies — this is disclosed in the component's own code comment, not something to strip out. Keep that attribution comment intact if this is reused, since it credits the actual author of the underlying visualization technique.
- **Summary**: Renders an interactive rotating globe made entirely of monospaced text — a hidden phrase is spelled out letter-by-letter across landmasses (determined by an embedded base64-encoded land/sea bitmap), while ocean and other land points render as small dot glyphs. The globe auto-idles with a slow spin, responds to drag-to-rotate and scroll-to-zoom, and clicking places small ping/pulse markers ("pins") on the surface that fade over time. Ships with dark/light theming built into both the outer React wrapper and the embedded HTML's own CSS variables, plus a `GlobeStudy` React prop layer for `scale`/`opacity`/hue-rotate/saturation/brightness post-processing via CSS `filter`.
- **Notes**: Architecturally very different from every other stashed component — really "a vanilla JS art piece wrapped in an iframe for React interop," not a typical composable UI component. Worth flagging: (1) `sandbox="allow-scripts"` on the iframe means it can't reach the parent page's DOM/state — treat it as a fully isolated visual widget, not something to wire to app state or theming beyond the `mode`/color props already exposed; (2) because it's an iframe, it won't inherit the project's Tailwind/CSS design tokens automatically — the dark/light color values are hardcoded inside the embedded HTML's own `<style>` block, so matching it to a project's theme means editing those hardcoded hex/rgba values inside the string, not just passing Tailwind classes; (3) it's a heavy, showy centerpiece — best suited to a single hero/about-page moment, not a repeated pattern across many sections. Good candidate for a "why work with us"/manifesto/about-page hero rather than a generic decorative background. Because the embedded source is long (~400 lines of canvas math), when this gets used, fetch or reuse the component's full original file rather than reconstructing the canvas logic from a description — this stash entry keeps the wrapper code complete but abbreviates the embedded HTML/JS for length; don't invent globe rendering logic from scratch.

```tsx
import { useMemo, type CSSProperties } from "react";

// Verbatim (trimmed to the 'globe' study) from MengTo/threeui, MIT licensed:
// src/shaders/text-path-studies/sources/text-on-a-path-ii.html
//
// NOTE: the full embedded HTML/JS string (canvas setup, land/sea bitmap,
// globe projection math, drag/zoom/pin interaction handlers, and the
// requestAnimationFrame render loop — several hundred lines) is abbreviated
// here for length. Keep the original complete GLOBE_STUDY_SOURCE string
// intact when actually using this component.
const GLOBE_STUDY_SOURCE = `<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Text on a Path II — Globe</title>
<style>
  :root{
    --bg:#08090a;
    --line:rgba(255,255,255,.028);
    --fig:rgba(255,255,255,.24);
    --title:#f2f3f5;
    --copy:rgba(255,255,255,.46);
  }
  /* ...full study CSS omitted for brevity... */
</style>
</head>
<body>
  <!-- canvas + interaction script rendering the rotating text globe -->
</body>
</html>
`;

export type GlobeStudyProps = {
  mode?: "dark" | "light";
  scale?: number;
  opacity?: number;
  hue?: number;
  saturation?: number;
  brightness?: number;
  className?: string;
  style?: CSSProperties;
};

export const GLOBE_STUDY_DEFAULTS = {
  mode: "dark",
  scale: 1,
  opacity: 1,
  hue: 0,
  saturation: 1,
  brightness: 1,
} as const;

function clamp(value: number, minimum: number, maximum: number) {
  return Math.min(maximum, Math.max(minimum, value));
}

function focusStyles(mode: "dark" | "light") {
  const surface = mode === "light" ? "#f3f5f8" : "#08090a";
  const themeStyles =
    mode === "light"
      ? `
      :root {
        color-scheme: light;
        --bg: #f3f5f8;
        --line: rgba(20, 24, 32, .055);
        --fig: rgba(20, 24, 32, .42);
        --title: #171922;
        --copy: rgba(20, 24, 32, .62);
      }
    `
      : ":root { color-scheme: dark; }";

  return `<style id="threeui-study-focus">
    ${themeStyles}
    html, body, .frame { width: 100% !important; height: 100% !important; overflow: hidden !important; }
    body { margin: 0 !important; background: ${surface} !important; }
    header, .fig h3, .fig p, .fignum { display: none !important; }
    .grid { display: block !important; width: 100% !important; height: 100% !important; }
    .fig { display: none !important; }
    .fig:nth-child(1) { display: flex !important; width: 100% !important; height: 100% !important; padding: 0 !important; }
    .art { display: flex !important; width: 100% !important; height: 100% !important; margin: 0 !important; align-items: center; justify-content: center; }
    .plate { width: min(100cqw, 100cqh) !important; height: min(100cqw, 100cqh) !important; }
  </style>`;
}

function focusedDocument(mode: "dark" | "light") {
  const source = mode === "light"
    ? GLOBE_STUDY_SOURCE.replace("var INK  = '226,228,233';", "var INK  = '38,40,48';")
    : GLOBE_STUDY_SOURCE;

  return source
    .replace(/<title>[\s\S]*?<\/title>/i, `<title>Globe — ThreeUI</title>`)
    .replace("</head>", `${focusStyles(mode)}\n</head>`);
}

export default function GlobeStudy({
  mode = GLOBE_STUDY_DEFAULTS.mode,
  scale = GLOBE_STUDY_DEFAULTS.scale,
  opacity = GLOBE_STUDY_DEFAULTS.opacity,
  hue = GLOBE_STUDY_DEFAULTS.hue,
  saturation = GLOBE_STUDY_DEFAULTS.saturation,
  brightness = GLOBE_STUDY_DEFAULTS.brightness,
  className,
  style,
}: GlobeStudyProps) {
  const safeMode = mode === "light" ? "light" : "dark";
  const document = useMemo(() => focusedDocument(safeMode), [safeMode]);
  const boundedScale = clamp(scale, 0.65, 1.5);
  const boundedOpacity = clamp(opacity, 0.1, 1);
  const boundedHue = clamp(hue, -180, 180);
  const boundedSaturation = clamp(saturation, 0, 2);
  const boundedBrightness = clamp(brightness, 0.4, 1.8);
  const filter =
    boundedHue === 0 && boundedSaturation === 1 && boundedBrightness === 1
      ? undefined
      : `hue-rotate(${boundedHue}deg) saturate(${boundedSaturation}) brightness(${boundedBrightness})`;

  return (
    <div
      className={["text-path-study", `text-path-study--${safeMode}`, className].filter(Boolean).join(" ")}
      data-mode={safeMode}
      style={{ opacity: boundedOpacity, filter, width: "100%", height: "100%", ...style }}
    >
      <iframe
        className="text-path-study-frame"
        data-mode={safeMode}
        title="Globe interactive canvas study"
        sandbox="allow-scripts"
        srcDoc={document}
        style={{
          width: "100%",
          height: "100%",
          border: "none",
          display: "block",
          transform: boundedScale === 1 ? undefined : `scale(${boundedScale})`,
        }}
      />
    </div>
  );
}
```

**Demo usage:**
```tsx
import GlobeStudy from "@/components/ui/globe-study";

const settings = {
  mode: "dark",
  scale: 1,
  opacity: 1,
  hue: 0,
  saturation: 1,
  brightness: 1,
};

export default function Demo(props: Partial<typeof settings>) {
  const s = { ...settings, ...props };
  return (
    <div className="h-screen w-full">
      <GlobeStudy {...(s as any)} />
    </div>
  );
}
```

---

## Dot Border Button
- **Category**: button / hover micro-interaction — small, isolated effect (same "iframe study" architecture as Globe Study above)
- **Stack**: React wrapper around a fully self-contained plain-HTML/CSS button, rendered inside a sandboxed `<iframe srcDoc={...}>` — same pattern as the Globe Study entry: only the outer wrapper (`mode`/`hue`/`saturation`/`brightness` props, an isolation script that hides everything except the target button, CSS filter post-processing) is real React/TypeScript. The actual button/animation is vanilla HTML + CSS (`:has()` selector-driven hover animations, no JS animation logic at all beyond the isolation bootstrap).
- **Deps**: none beyond React for the wrapper. The **embedded iframe HTML itself loads Tailwind via `<script src="https://cdn.tailwindcss.com">`** at runtime (the Tailwind Play CDN) — that's a live external network request every time this iframe renders, separate from and in addition to the project's own build-time Tailwind. It's sandboxed inside the iframe so it can't affect the parent app's styles, but it's still a runtime CDN dependency worth knowing is there — mention it before using this in production or an offline-capable app.
- **Source/license note**: Explicitly commented in the source as verbatim from **MengTo/threeui** (MIT licensed), from its "neuform-isolated" component studies — same attribution family as the Globe Study entry. Keep the attribution comment intact when reused.
- **Summary**: A button with a "measured/drafting" hover effect — on hover, four small square dots animate outward toward the corners while dashed border lines "draw themselves" inward from each edge (via staggered `scaleX`/`scaleY` keyframe animations with sequential `animation-delay`s), plus a diagonal grid-pattern overlay that fades in behind the button. Button itself has a simple scale + letter-spacing hover/active state and ships with a default "Start Creating" label and an arrow/pencil-style SVG icon. Fully theme-able via CSS custom properties already exposed in the source (`--dot-size`, `--line-weight`, `--line-distance`, `--animation-speed`, `--dot-color`, `--line-color`, `--grid-color`) — these can be tweaked without touching the animation logic itself.
- **Notes**: Uses the CSS `:has()` selector for hover-state chaining (`.btn-wrapper:has(.btn:hover)`) — this is broadly supported in modern evergreen browsers but not in older ones; check the project's browser support target before relying on it. Because it's isolated in a sandboxed iframe like Globe Study, this is a lot of architectural overhead (a whole loaded HTML document + Tailwind CDN fetch + isolation script) for what is fundamentally a single CSS-only button effect — if the project doesn't already have other iframe-based "studies" from this same source, it's usually simpler to just port the raw CSS (the `.btn-wrapper`/`.dot`/`.line` rules and keyframes) directly into the project's own stylesheet/component rather than keeping the iframe wrapper, unless isolation from the parent page's styles is specifically wanted. The placeholder `href="#"` click is already prevented inside the embedded script, so it's non-navigating out of the box — swap in a real `onClick`/href by editing the embedded HTML's button markup, since props don't currently expose a way to pass a click handler through the iframe boundary.

```tsx
import { useMemo, type CSSProperties } from "react";

type FocusRole = "background" | "button" | "visual";
type EffectMode = "light" | "dark";

type FocusTarget = {
  selector: string;
  role: FocusRole;
  preserveTransform?: boolean;
};

type EffectDefinition = {
  title: string;
  source: string;
  background: string;
  targets: readonly FocusTarget[];
  theme?: {
    lightBackground: string;
    darkBackground: string;
  };
};

export type DotBorderButtonProps = {
  mode?: EffectMode;
  hue?: number;
  saturation?: number;
  brightness?: number;
  className?: string;
  style?: CSSProperties;
};

const DOT_BORDER_BUTTON_DEFAULTS = {
  mode: "dark",
  hue: 0,
  saturation: 1,
  brightness: 1,
} as const;

// Verbatim from MengTo/threeui:
// src/shaders/neuform-isolated/sources/dot-border-button.html
const DOT_BORDER_BUTTON_SOURCE = `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Component Preview</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    html, body { height: 100%; margin: 0; padding: 0; }
    body {
      height: 100%;
      overflow: auto;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: #000000;
      color: #ffffff;
    }
    .component-wrapper { width: 100%; height: 100%; padding: 0; box-sizing: border-box; overflow: auto; }
  </style>
</head>
<body>
  <div class="component-wrapper">
    <div style="display: flex; justify-content: center; align-items: center; height: 100vh; background-color: #000;">
      <a href="#" class="btn-wrapper" style="--dot-size: 8px; --line-weight: 1px; --line-distance: 0.8rem 1rem; --animation-speed: 0.35s; --dot-color: #fffa; --line-color: #fffa; --grid-color: #fff3; position: relative; display: inline-flex; justify-content: center; align-items: center; width: auto; height: auto; padding: var(--line-distance); background-color: rgba(0, 0, 0, 0); user-select: none">
        <style>
          .btn-wrapper::after {
            content: ""; position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            border-radius: inherit; pointer-events: none; background-color: #0000;
            background-image: repeating-linear-gradient(45deg, var(--grid-color) 0 1px, transparent 2px 5px);
            opacity: 0; z-index: -1;
          }
          .btn-wrapper:has(.btn:hover)::after { animation: opacity-anim calc(var(--animation-speed) * 4) ease-in-out forwards; }
          @keyframes opacity-anim { 80% { opacity: 0; } 100% { opacity: 1; } }

          .btn-wrapper .btn {
            position: relative; display: flex; justify-content: center; align-items: center;
            padding: 0.8rem 1.25rem; background-color: #fff0; border: 1px solid var(--grid-color);
            color: #fffd; font-family: "Inter", sans-serif; letter-spacing: -0.01em;
            font-size: 1rem; font-weight: 600; text-transform: capitalize; border-radius: 6px;
            cursor: pointer; transition: transform .2s ease-in-out, letter-spacing .2s ease-in-out;
          }
          .btn-wrapper .btn:hover { background-color: #25358b; color: #fff; transform: scale(1.05); letter-spacing: .06em; }
          .btn-wrapper .btn:active { background-color: #25358b; transform: scale(.98); letter-spacing: .02em; }
          .btn-wrapper .btn-svg { margin-left: .5rem; height: 24px; stroke-width: 1; stroke-linecap: round; stroke-linejoin: round; stroke: #fff4; fill: #fff2; transition: all .2s ease-in-out; }
          .btn-wrapper .btn:hover .btn-svg { stroke: #fffa; fill: #fff3; }

          .btn-wrapper .dot { position: absolute; width: var(--dot-size); aspect-ratio: 1; border-radius: 2px; background-color: var(--dot-color); transition: all .3s ease-in-out; opacity: 0; }
          .btn-wrapper:has(.btn:hover) .dot.top.left { top: 50%; left: 20%; animation: move-top-left var(--animation-speed) ease-in-out forwards; }
          @keyframes move-top-left { 90% { opacity: .6; } 100% { top: calc(var(--dot-size) * -0.5); left: calc(var(--dot-size) * -0.5); opacity: 1; } }
          .btn-wrapper:has(.btn:hover) .dot.top.right { top: 50%; right: 20%; animation: move-top-right var(--animation-speed) ease-in-out forwards; animation-delay: calc(var(--animation-speed)*.6); }
          @keyframes move-top-right { 80% { opacity: .6; } 100% { top: calc(var(--dot-size) * -0.5); right: calc(var(--dot-size) * -0.5); opacity: 1; } }
          .btn-wrapper:has(.btn:hover) .dot.bottom.right { bottom: 50%; right: 20%; animation: move-bottom-right var(--animation-speed) ease-in-out forwards; animation-delay: calc(var(--animation-speed)*1.2); }
          @keyframes move-bottom-right { 80% { opacity: .6; } 100% { bottom: calc(var(--dot-size) * -0.5); right: calc(var(--dot-size) * -0.5); opacity: 1; } }
          .btn-wrapper:has(.btn:hover) .dot.bottom.left { bottom: 50%; left: 20%; animation: move-bottom-left var(--animation-speed) ease-in-out forwards; animation-delay: calc(var(--animation-speed)*1.8); }
          @keyframes move-bottom-left { 80% { opacity: .6; } 100% { bottom: calc(var(--dot-size) * -0.5); left: calc(var(--dot-size) * -0.5); opacity: 1; } }

          .btn-wrapper .line { position: absolute; transition: all .3s ease-in-out; }
          .btn-wrapper .line.horizontal { height: var(--line-weight); width: 100%; background-image: repeating-linear-gradient(90deg, #0000 0 calc(var(--line-weight)*2), var(--line-color) calc(var(--line-weight)*2) calc(var(--line-weight)*4)); }
          .btn-wrapper .line.top { top: calc(var(--line-weight)*-0.5); transform-origin: top left; transform: rotate(5deg) scaleX(0); }
          .btn-wrapper:has(.btn:hover) .line.top { animation: draw-top var(--animation-speed) ease-in-out forwards; animation-delay: calc(var(--animation-speed)*.8); }
          @keyframes draw-top { 100% { transform: rotate(0deg) scaleX(1); } }
          .btn-wrapper .line.bottom { bottom: calc(var(--line-weight)*-0.5); transform-origin: bottom right; transform: rotate(5deg) scaleX(0); }
          .btn-wrapper:has(.btn:hover) .line.bottom { animation: draw-bottom var(--animation-speed) ease-in-out forwards; animation-delay: calc(var(--animation-speed)*2); }
          @keyframes draw-bottom { 100% { transform: rotate(0deg) scaleX(1); } }
          .btn-wrapper .line.vertical { width: var(--line-weight); height: 100%; background-image: repeating-linear-gradient(0deg, #0000 0 calc(var(--line-weight)*2), var(--line-color) calc(var(--line-weight)*2) calc(var(--line-weight)*4)); }
          .btn-wrapper .line.left { left: calc(var(--line-weight)*-0.5); transform-origin: bottom left; transform: rotate(0deg) scaleY(0); }
          .btn-wrapper:has(.btn:hover) .line.left { animation: draw-left var(--animation-speed) ease-in-out forwards; animation-delay: calc(var(--animation-speed)*2.4); }
          @keyframes draw-left { 100% { transform: rotate(0deg) scaleY(1); } }
          .btn-wrapper .line.right { right: calc(var(--line-weight)*-0.5); transform-origin: top right; transform: rotate(5deg) scaleY(0); }
          .btn-wrapper:has(.btn:hover) .line.right { animation: draw-right var(--animation-speed) ease-in-out forwards; animation-delay: calc(var(--animation-speed)*1.4); }
          @keyframes draw-right { 100% { transform: rotate(0deg) scaleY(1); } }
        </style>

        <div class="line horizontal top"></div>
        <div class="line vertical right"></div>
        <div class="line horizontal bottom"></div>
        <div class="line vertical left"></div>

        <div class="dot top left"></div>
        <div class="dot top right"></div>
        <div class="dot bottom right"></div>
        <div class="dot bottom left"></div>

        <button class="btn bg-[#ffffff]">
          <span class="btn-text">Start Creating</span>
          <svg class="btn-svg" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
            <path d="M17.6744 11.4075L15.7691 17.1233C15.7072 17.309 15.5586 17.4529 15.3709 17.5087L3.69348 20.9803C3.22819 21.1186 2.79978 20.676 2.95328 20.2155L6.74467 8.84131C6.79981 8.67588 6.92419 8.54263 7.08543 8.47624L12.472 6.25822C12.696 6.166 12.9535 6.21749 13.1248 6.38876L17.5294 10.7935C17.6901 10.9542 17.7463 11.1919 17.6744 11.4075Z"></path>
            <path d="M3.2959 20.6016L9.65986 14.2376"></path>
            <path d="M17.7917 11.0557L20.6202 8.22724C21.4012 7.44619 21.4012 6.17986 20.6202 5.39881L18.4989 3.27749C17.7178 2.49645 16.4515 2.49645 15.6704 3.27749L12.842 6.10592"></path>
            <path d="M11.7814 12.1163C11.1956 11.5305 10.2458 11.5305 9.66004 12.1163C9.07426 12.7021 9.07426 13.6519 9.66004 14.2376C10.2458 14.8234 11.1956 14.8234 11.7814 14.2376C12.3671 13.6519 12.3671 12.7021 11.7814 12.1163Z"></path>
          </svg>
        </button>
      </a>
    </div>
  </div>
</body>
</html>
`;

const DOT_BORDER_BUTTON_DEFINITION: EffectDefinition = {
  title: "Dot Border button",
  source: DOT_BORDER_BUTTON_SOURCE,
  background: "#111318",
  theme: { lightBackground: "#f4f7fb", darkBackground: "#111318" },
  targets: [
    { selector: ".component-wrapper .btn-wrapper", role: "button", preserveTransform: true },
  ],
};

function clamp(value: number, minimum: number, maximum: number) {
  return Math.min(maximum, Math.max(minimum, value));
}

function effectBackground(definition: EffectDefinition, mode: EffectMode) {
  return definition.theme?.[`${mode}Background`] ?? definition.background;
}

function buildFocusedDocument(definition: EffectDefinition, mode: EffectMode) {
  const background = effectBackground(definition, mode);
  const targetJson = JSON.stringify(definition.targets).replace(/</g, "\\u003c");
  const modeJson = JSON.stringify(mode);
  const focusStyle = `<style data-threeui-focus>
html, body { width: 100% !important; height: 100% !important; min-height: 0 !important; margin: 0 !important; padding: 0 !important; overflow: hidden !important; background: ${background} !important; color-scheme: ${mode} !important; }
body { position: relative !important; display: flex !important; align-items: center !important; justify-content: center !important; }
body > * { visibility: hidden !important; }
body[data-threeui-ready] > [data-threeui-role] { visibility: visible !important; }
[data-threeui-residual] { display: none !important; }
[data-threeui-role="button"] { position: relative !important; z-index: 2 !important; opacity: 1 !important; flex: none !important; }
[data-threeui-role="button"]:not([data-threeui-preserve-transform]) { transform: none !important; }
</style>`;
  const focusScript = `<script data-threeui-focus>
(function () {
  document.documentElement.dataset.sfMode = ${modeJson};
  var isolated = false;
  function isolate() {
    if (isolated) return;
    var specs = ${targetJson};
    var roots = [];
    specs.forEach(function (spec) {
      var element = document.querySelector(spec.selector);
      if (!element) return;
      element.setAttribute('data-threeui-role', spec.role);
      if (spec.preserveTransform) element.setAttribute('data-threeui-preserve-transform', '');
      if (!roots.some(function (root) { return root.contains(element); })) roots.push(element);
    });
    if (!roots.length) return;
    isolated = true;
    roots.forEach(function (root) {
      var placeholderLink = root.matches('a[href="#"]') ? root : root.querySelector('a[href="#"]');
      if (placeholderLink) placeholderLink.addEventListener('click', function (event) { event.preventDefault(); });
      document.body.appendChild(root);
    });
    Array.from(document.body.children).forEach(function (element) {
      if (roots.indexOf(element) !== -1) return;
      element.setAttribute('data-threeui-residual', '');
      element.setAttribute('aria-hidden', 'true');
      if ('inert' in element) element.inert = true;
    });
    document.body.setAttribute('data-threeui-ready', '');
    requestAnimationFrame(function () { window.dispatchEvent(new Event('resize')); });
  }
  function scheduleIsolation() { setTimeout(isolate, 100); }
  if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', scheduleIsolation, { once: true });
  else scheduleIsolation();
  window.addEventListener('load', isolate, { once: true });
})();
</script>`;
  return definition.source
    .replace(/<\/head>/i, `${focusStyle}</head>`)
    .replace(/<\/body>/i, `${focusScript}</body>`);
}

function NeuformIsolatedEffect({
  definition,
  mode = DOT_BORDER_BUTTON_DEFAULTS.mode,
  hue = DOT_BORDER_BUTTON_DEFAULTS.hue,
  saturation = DOT_BORDER_BUTTON_DEFAULTS.saturation,
  brightness = DOT_BORDER_BUTTON_DEFAULTS.brightness,
  className,
  style,
}: DotBorderButtonProps & { definition: EffectDefinition }) {
  const safeMode: EffectMode = mode === "light" ? "light" : "dark";
  const background = effectBackground(definition, safeMode);
  const source = useMemo(() => buildFocusedDocument(definition, safeMode), [definition, safeMode]);
  const safeHue = clamp(hue, -180, 180);
  const safeSaturation = clamp(saturation, 0, 2);
  const safeBrightness = clamp(brightness, 0.35, 1.65);
  const filter =
    safeHue === 0 && safeSaturation === 1 && safeBrightness === 1
      ? undefined
      : `hue-rotate(${safeHue}deg) saturate(${safeSaturation}) brightness(${safeBrightness})`;

  return (
    <iframe
      className={className}
      data-mode={safeMode}
      title={definition.title}
      srcDoc={source}
      sandbox="allow-scripts"
      loading="eager"
      style={{ display: "block", width: "100%", height: "100%", border: 0, background, filter, ...style }}
    />
  );
}

function DotBorderButton(props: DotBorderButtonProps) {
  return <NeuformIsolatedEffect {...props} definition={DOT_BORDER_BUTTON_DEFINITION} />;
}

export default DotBorderButton;
```

**Demo usage:**
```tsx
import DotBorderButton from "@/components/ui/dot-border-button";

export default function DotBorderButtonDemo() {
  return (
    <div className="flex h-[420px] w-full items-center justify-center overflow-hidden rounded-xl border border-border bg-[#111318]">
      <DotBorderButton mode="dark" className="h-full w-full" />
    </div>
  );
}
```

---

## macOS Dock
- **Category**: navigation dock / app launcher — desktop-OS-style UI chrome
- **Stack**: React, TypeScript, inline styles (no Tailwind, no CSS-in-JS library) — pure `requestAnimationFrame`-driven imperative animation, no `framer-motion`/`motion` dependency at all
- **Deps**: none required. Has an **optional** runtime check for a global `window.gsap` (`if (typeof window !== 'undefined' && (window as any).gsap)`) to use GSAP's bounce animation on click if GSAP happens to already be loaded on the page; if it's not present, it gracefully falls back to a manual CSS-transition bounce (`createBounceAnimation`) — so unlike the CDN-injection components flagged elsewhere in this stash, this one doesn't load anything itself, it just opportunistically uses GSAP if the project already has it. Nothing to install unless GSAP-quality bounce easing is specifically wanted, in which case add `gsap` as areal dependency and import it directly rather than relying on a global.
- **Summary**: A pixel-accurate recreation of macOS's Dock magnification effect — icons scale up in a smooth cosine falloff curve as the cursor approaches (authentic to Apple's actual algorithm, not just a generic hover-scale), with neighboring icons scaling proportionally less the further they are from the cursor, and everything lerped frame-by-frame via `requestAnimationFrame` for buttery interpolation rather than CSS transitions on hover. Responsive: recalculates icon size/max-scale/effect-radius based on viewport size (distinct tuned configs for phone/tablet/small-laptop/desktop breakpoints) rather than fixed Tailwind breakpoints. Includes running-app indicator dots and a translucent frosted-glass dock background with layered box-shadows for depth.
- **Notes**: The demo's icon URLs point at 21st.dev's own asset CDN (`cdn.21st.dev/assets/mirror/...`) — these are 21st.dev's hosted mirror of macOS system icons for demo purposes, not something to keep pointing at in a real project (that CDN's availability/uptime isn't guaranteed for external use, and reusing macOS's actual app icons in a shipped product raises the same trademark/likeness consideration as the Floating Icons Hero's brand logos — macOS's Finder/Safari/etc. icons are Apple's IP). Swap in the user's own app icons/images before using this for anything beyond a demo or portfolio piece imitating macOS. The click bounce and magnification math both use fairly involved manual physics (cosine falloff, lerp factors, throttled mousemove at ~60fps) — this is a case where the "adapt, don't just paste" step mostly means retheming the dock's colors/blur/shadow to match the project rather than touching the animation math, which is already well-tuned; avoid "simplifying" the cosine calculation, it's what makes the effect feel authentic rather than like a generic hover-scale.

```tsx
'use client';

import React, { useState, useRef, useCallback, useEffect } from 'react';

interface DockApp {
  id: string;
  name: string;
  icon: string;
}

interface MacOSDockProps {
  apps: DockApp[];
  onAppClick: (appId: string) => void;
  openApps?: string[];
  className?: string;
}

const MacOSDock: React.FC<MacOSDockProps> = ({ 
  apps, 
  onAppClick, 
  openApps = [],
  className = ''
}) => {
  const [mouseX, setMouseX] = useState<number | null>(null);
  const [currentScales, setCurrentScales] = useState<number[]>(apps.map(() => 1));
  const [currentPositions, setCurrentPositions] = useState<number[]>([]);
  const dockRef = useRef<HTMLDivElement>(null);
  const iconRefs = useRef<(HTMLDivElement | null)[]>([]);
  const animationFrameRef = useRef<number | undefined>(undefined);
  const lastMouseMoveTime = useRef<number>(0);

  const getResponsiveConfig = useCallback(() => {
    if (typeof window === 'undefined') {
      return { baseIconSize: 64, maxScale: 1.6, effectWidth: 240 };
    }

    const smallerDimension = Math.min(window.innerWidth, window.innerHeight);
    
    if (smallerDimension < 480) {
      return {
        baseIconSize: Math.max(40, smallerDimension * 0.08),
        maxScale: 1.4,
        effectWidth: smallerDimension * 0.4
      };
    } else if (smallerDimension < 768) {
      return {
        baseIconSize: Math.max(48, smallerDimension * 0.07),
        maxScale: 1.5,
        effectWidth: smallerDimension * 0.35
      };
    } else if (smallerDimension < 1024) {
      return {
        baseIconSize: Math.max(56, smallerDimension * 0.06),
        maxScale: 1.6,
        effectWidth: smallerDimension * 0.3
      };
    } else {
      return {
        baseIconSize: Math.max(64, Math.min(80, smallerDimension * 0.05)),
        maxScale: 1.8,
        effectWidth: 300
      };
    }
  }, []);

  const [config, setConfig] = useState(getResponsiveConfig);
  const { baseIconSize, maxScale, effectWidth } = config;
  const minScale = 1.0;
  const baseSpacing = Math.max(4, baseIconSize * 0.08);

  useEffect(() => {
    const handleResize = () => {
      setConfig(getResponsiveConfig());
    };

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, [getResponsiveConfig]);

  const calculateTargetMagnification = useCallback((mousePosition: number | null) => {
    if (mousePosition === null) {
      return apps.map(() => minScale);
    }

    return apps.map((_, index) => {
      const normalIconCenter = (index * (baseIconSize + baseSpacing)) + (baseIconSize / 2);
      const minX = mousePosition - (effectWidth / 2);
      const maxX = mousePosition + (effectWidth / 2);
      
      if (normalIconCenter < minX || normalIconCenter > maxX) {
        return minScale;
      }
      
      const theta = ((normalIconCenter - minX) / effectWidth) * 2 * Math.PI;
      const cappedTheta = Math.min(Math.max(theta, 0), 2 * Math.PI);
      const scaleFactor = (1 - Math.cos(cappedTheta)) / 2;
      
      return minScale + (scaleFactor * (maxScale - minScale));
    });
  }, [apps, baseIconSize, baseSpacing, effectWidth, maxScale, minScale]);

  const calculatePositions = useCallback((scales: number[]) => {
    let currentX = 0;
    
    return scales.map((scale) => {
      const scaledWidth = baseIconSize * scale;
      const centerX = currentX + (scaledWidth / 2);
      currentX += scaledWidth + baseSpacing;
      return centerX;
    });
  }, [baseIconSize, baseSpacing]);

  useEffect(() => {
    const initialScales = apps.map(() => minScale);
    const initialPositions = calculatePositions(initialScales);
    setCurrentScales(initialScales);
    setCurrentPositions(initialPositions);
  }, [apps, calculatePositions, minScale, config]);

  const animateToTarget = useCallback(() => {
    const targetScales = calculateTargetMagnification(mouseX);
    const targetPositions = calculatePositions(targetScales);
    const lerpFactor = mouseX !== null ? 0.2 : 0.12;

    setCurrentScales(prevScales => {
      return prevScales.map((currentScale, index) => {
        const diff = targetScales[index] - currentScale;
        return currentScale + (diff * lerpFactor);
      });
    });

    setCurrentPositions(prevPositions => {
      return prevPositions.map((currentPos, index) => {
        const diff = targetPositions[index] - currentPos;
        return currentPos + (diff * lerpFactor);
      });
    });

    const scalesNeedUpdate = currentScales.some((scale, index) => 
      Math.abs(scale - targetScales[index]) > 0.002
    );
    const positionsNeedUpdate = currentPositions.some((pos, index) => 
      Math.abs(pos - targetPositions[index]) > 0.1
    );
    
    if (scalesNeedUpdate || positionsNeedUpdate || mouseX !== null) {
      animationFrameRef.current = requestAnimationFrame(animateToTarget);
    }
  }, [mouseX, calculateTargetMagnification, calculatePositions, currentScales, currentPositions]);

  useEffect(() => {
    if (animationFrameRef.current) {
      cancelAnimationFrame(animationFrameRef.current);
    }
    animationFrameRef.current = requestAnimationFrame(animateToTarget);

    return () => {
      if (animationFrameRef.current) {
        cancelAnimationFrame(animationFrameRef.current);
      }
    };
  }, [animateToTarget]);

  const handleMouseMove = useCallback((e: React.MouseEvent) => {
    const now = performance.now();
    
    if (now - lastMouseMoveTime.current < 16) {
      return;
    }
    
    lastMouseMoveTime.current = now;
    
    if (dockRef.current) {
      const rect = dockRef.current.getBoundingClientRect();
      const padding = Math.max(8, baseIconSize * 0.12);
      setMouseX(e.clientX - rect.left - padding);
    }
  }, [baseIconSize]);

  const handleMouseLeave = useCallback(() => {
    setMouseX(null);
  }, []);

  const createBounceAnimation = (element: HTMLElement) => {
    const bounceHeight = Math.max(-8, -baseIconSize * 0.15);
    element.style.transition = 'transform 0.2s ease-out';
    element.style.transform = `translateY(${bounceHeight}px)`;
    
    setTimeout(() => {
      element.style.transform = 'translateY(0px)';
    }, 200);
  };

  const handleAppClick = (appId: string, index: number) => {
    if (iconRefs.current[index]) {
      if (typeof window !== 'undefined' && (window as any).gsap) {
        const gsap = (window as any).gsap;
        const bounceHeight = currentScales[index] > 1.3 ? -baseIconSize * 0.2 : -baseIconSize * 0.15;
        
        gsap.to(iconRefs.current[index], {
          y: bounceHeight,
          duration: 0.2,
          ease: 'power2.out',
          yoyo: true,
          repeat: 1,
          transformOrigin: 'bottom center'
        });
      } else {
        createBounceAnimation(iconRefs.current[index]!);
      }
    }
    
    onAppClick(appId);
  };

  const contentWidth = currentPositions.length > 0 
    ? Math.max(...currentPositions.map((pos, index) => 
        pos + (baseIconSize * currentScales[index]) / 2
      ))
    : (apps.length * (baseIconSize + baseSpacing)) - baseSpacing;

  const padding = Math.max(8, baseIconSize * 0.12);

    return (
    <div 
      ref={dockRef}
      className={`backdrop-blur-md ${className}`}
      style={{
        width: `${contentWidth + padding * 2}px`,
        background: 'rgba(45, 45, 45, 0.75)',
        borderRadius: `${Math.max(12, baseIconSize * 0.4)}px`,
        border: '1px solid rgba(255, 255, 255, 0.15)',
        boxShadow: `
          0 ${Math.max(4, baseIconSize * 0.1)}px ${Math.max(16, baseIconSize * 0.4)}px rgba(0, 0, 0, 0.4),
          0 ${Math.max(2, baseIconSize * 0.05)}px ${Math.max(8, baseIconSize * 0.2)}px rgba(0, 0, 0, 0.3),
          inset 0 1px 0 rgba(255, 255, 255, 0.15),
          inset 0 -1px 0 rgba(0, 0, 0, 0.2)
        `,
        padding: `${padding}px`
      }}
      onMouseMove={handleMouseMove}
      onMouseLeave={handleMouseLeave}
    >
      <div 
        className="relative"
        style={{
          height: `${baseIconSize}px`,
          width: '100%'
        }}
      >
        {apps.map((app, index) => {
          const scale = currentScales[index];
          const position = currentPositions[index] || 0;
          const scaledSize = baseIconSize * scale;
          
          return (
            <div
              key={app.id}
              ref={(el) => { iconRefs.current[index] = el; }}
              className="absolute cursor-pointer flex flex-col items-center justify-end"
              title={app.name}
              onClick={() => handleAppClick(app.id, index)}
              style={{
                left: `${position - scaledSize / 2}px`,
                bottom: '0px',
                width: `${scaledSize}px`,
                height: `${scaledSize}px`,
                transformOrigin: 'bottom center',
                zIndex: Math.round(scale * 10)
              }}
            >
              <img
                src={app.icon}
                alt={app.name}
                width={scaledSize}
                height={scaledSize}
                className="object-contain"
                style={{
                  filter: `drop-shadow(0 ${scale > 1.2 ? Math.max(2, baseIconSize * 0.05) : Math.max(1, baseIconSize * 0.03)}px ${scale > 1.2 ? Math.max(4, baseIconSize * 0.1) : Math.max(2, baseIconSize * 0.06)}px rgba(0,0,0,${0.2 + (scale - 1) * 0.15}))`
                }}
              />
              
              {openApps.includes(app.id) && (
                <div 
                  className="absolute"
                  style={{
                    bottom: `${Math.max(-2, -baseIconSize * 0.05)}px`,
                    left: '50%',
                    transform: 'translateX(-50%)',
                    width: `${Math.max(3, baseIconSize * 0.06)}px`,
                    height: `${Math.max(3, baseIconSize * 0.06)}px`,
                    borderRadius: '50%',
                    backgroundColor: 'rgba(255, 255, 255, 0.8)',
                    boxShadow: '0 0 4px rgba(0, 0, 0, 0.3)',
                  }}
                />
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
};

export default MacOSDock;
```

**Demo usage** (⚠️ icon URLs point at 21st.dev's demo asset CDN and depict real macOS app icons — swap for the user's own images before shipping):

```tsx
import React, { useState } from 'react';
import MacOSDock from './components/ui/mac-os-dock.tsx';

const sampleApps = [
  { id: 'finder', name: 'Finder', icon: '/* project's own icon path */' },
  { id: 'calculator', name: 'Calculator', icon: '/* project's own icon path */' },
  { id: 'terminal', name: 'Terminal', icon: '/* project's own icon path */' },
  // ...remaining apps follow the same { id, name, icon } shape
];

const DockDemo: React.FC = () => {
  const [openApps, setOpenApps] = useState<string[]>(['finder', 'safari']);

  const handleAppClick = (appId: string) => {
    setOpenApps(prev => 
      prev.includes(appId) 
        ? prev.filter(id => id !== appId)
        : [...prev, appId]
    );
  };

  return (
    <div style={{ height: '100vh', width: '100vw', display: 'flex', alignItems: 'center', justifyContent: 'center', overflow: 'hidden' }}>
      <MacOSDock apps={sampleApps} onAppClick={handleAppClick} openApps={openApps} />
    </div>
  );
};

export default DockDemo;
```

---

## Sticky Scroll Gallery
- **Category**: image gallery / scroll-driven layout — landing/portfolio section
- **Stack**: React, TypeScript, Tailwind, `lenis/react` (`ReactLenis`) for smooth-scroll — no other animation library, the visual effect is pure CSS `sticky` positioning, not JS-driven scroll animation
- **Deps**: `lenis` (the `ReactLenis` wrapper from the `lenis/react` subpath) — this replaces the browser's native scroll with Lenis's smoothed/eased scroll for the whole page (`root` prop), which is a global effect, not scoped to just this section
- **Summary**: A three-column image gallery where the center column is `sticky top-0 h-screen` while the left and right columns scroll normally — as the page scrolls, the two outer columns appear to slide past the fixed center column, creating a parallax-like "gallery wall" effect using nothing but CSS sticky positioning (no scroll-linked JS calculations). Preceded by a full-screen hero with a radial-masked dot-grid background and centered headline, and followed by a large uppercase gradient-text footer wordmark ("ui-layout" in the original — swap for the project's own name/logo).
- **Notes**: `<ReactLenis root>` swaps the smooth-scroll behavior for the **entire page**, not just this section — if the project already uses Lenis elsewhere, don't wrap it a second time (only one `root` Lenis instance should exist per page); if it doesn't, mounting this component will change scroll feel sitewide, which is worth confirming with the user is wanted rather than assuming. All 11 image URLs point at 21st.dev's demo asset CDN (`cdn.21st.dev/assets/mirror/...`) — swap for the user's own gallery images before using this beyond a demo (same caveat as the macOS Dock's icon URLs: that CDN mirror isn't meant for external production use). The center column is hardcoded to exactly 3 images filling a `grid-rows-3` — if the user wants a different image count in the sticky column, that grid needs adjusting, it won't auto-flow. The footer wordmark literally says "ui-layout" (likely because this was itself sourced from the ui-layouts.com library, already in `references/libraries.md`) — that needs to be swapped to the project's own brand name, not left as attribution to the source library.

```tsx
'use client';
import { ReactLenis } from 'lenis/react';
import React, { forwardRef } from 'react';

const Component = forwardRef<HTMLElement>((props, ref) => {
  return (
    <ReactLenis root>
      <main className='bg-black' ref={ref}>
        <div className='wrapper'>
          <section className='text-white  h-screen  w-full bg-slate-950  grid place-content-center sticky top-0'>
            <div className='absolute bottom-0 left-0 right-0 top-0 bg-[linear-gradient(to_right,#4f4f4f2e_1px,transparent_1px),linear-gradient(to_bottom,#4f4f4f2e_1px,transparent_1px)] bg-[size:54px_54px] [mask-image:radial-gradient(ellipse_60%_50%_at_50%_0%,#000_70%,transparent_100%)]'></div>

            <h1 className='2xl:text-7xl text-5xl px-8 font-semibold text-center tracking-tight leading-[120%]'>
              Create Gallery In a Better Way
              <br />
              Using CSS sticky properties <br />
              Scroll down! 👇
            </h1>
          </section>
        </div>

        <section className='text-white   w-full bg-slate-950  '>
          <div className='grid grid-cols-12 gap-2'>
            <div className='grid gap-2 col-span-4'>
              <figure className=' w-full'>
                <img src='/* project image 1 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className=' w-full'>
                <img src='/* project image 2 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className=' w-full'>
                <img src='/* project image 3 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full'>
                <img src='/* project image 4 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full'>
                <img src='/* project image 5 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
            </div>
            <div className='sticky top-0 h-screen w-full col-span-4 gap-2  grid grid-rows-3'>
              <figure className='w-full h-full '>
                <img src='/* sticky column image 1 */' alt='' className='transition-all duration-300 h-full w-full align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full h-full '>
                <img src='/* sticky column image 2 */' alt='' className='transition-all duration-300 h-full w-full align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full h-full '>
                <img src='/* sticky column image 3 */' alt='' className='transition-all duration-300 h-full w-full align-bottom object-cover rounded-md' />
              </figure>
            </div>
            <div className='grid gap-2 col-span-4'>
              <figure className='w-full'>
                <img src='/* project image 6 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full'>
                <img src='/* project image 7 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full'>
                <img src='/* project image 8 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full'>
                <img src='/* project image 9 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
              <figure className='w-full'>
                <img src='/* project image 10 */' alt='' className='transition-all duration-300 w-full h-96 align-bottom object-cover rounded-md' />
              </figure>
            </div>
          </div>
        </section>

        <footer className='group bg-slate-950 '>
          <h1 className='text-[16vw]  translate-y-20 leading-[100%] uppercase font-semibold text-center bg-gradient-to-r from-gray-400 to-gray-800 bg-clip-text text-transparent transition-all ease-linear'>
            {/* swap this wordmark for the project's own name/logo */}
            your-brand
          </h1>
          <div className='bg-black h-40 relative z-10 grid place-content-center text-2xl rounded-tr-full rounded-tl-full'></div>
        </footer>
      </main>
    </ReactLenis>
  );
});

Component.displayName = 'Component';

export default Component;
```

**Demo usage:**
```tsx
import React from 'react';
import Component from '@/components/ui/sticky-scroll';

function ComponentDemo() {
  return (
    <Component />
  );
}

export { ComponentDemo as DemoOne };
```

---

## CoverFlow Carousel
- **Category**: carousel / product-menu showcase — Apple-CoverFlow-style 3D card carousel
- **Stack**: React, TypeScript, plain inline styles (no Tailwind for the card visuals, though the outer section uses a couple Tailwind utility classes) — zero external dependencies, including icons: the chevrons/arrow are hand-written inline SVG components rather than imported from `lucide-react` or any icon set
- **Deps**: none — fully self-contained
- **Summary**: A 3D "CoverFlow"-style carousel (à la old iTunes/iPod) showing 5 cards in view at once: a large centered active card plus two fading, scaled-down, rotated cards receding on each side via `rotateY`/`scale`/`translateX` transforms and a CSS `perspective`. Includes autoplay (pausable on hover), keyboard arrow navigation, touch swipe support, click-to-jump on any visible side card, pagination dots, and a blurred/darkened ambient background image derived from the current slide. Content overlay (tag, two-line title, description, CTA button) only fades in on the center card. Ships with a restaurant/menu-themed default dataset (`defaultDishes`) but is fully data-driven via the `items` prop and a `CarouselItem` type — swapping content is just passing a different `items` array, no need to touch the carousel logic itself.
- **Notes**: All 5 default image URLs point at 21st.dev's demo asset CDN (`cdn.21st.dev/assets/mirror/...`) — same caveat as other stashed components sourced from 21st.dev demos: swap for the user's own images before shipping, don't rely on that CDN mirror staying available. The 3D transform offsets (`translateX(285px)`, `rotateY(-24deg)`, etc.) are hardcoded pixel/degree values tuned for the fixed `330px`-wide card — if the card size changes, these offsets need to scale proportionally too, they won't auto-adjust (no responsive/container-query logic here, unlike e.g. the macOS Dock's viewport-aware sizing). Only shows 5 cards total (center + 2 each side) regardless of how many items are in the array — with more than 5 items the rest are present in the DOM but fully hidden (`opacity: 0`) until navigated to, which is fine functionally but worth knowing if the user expects to see more cards peeking at once. Default autoplay interval is 5000ms — adjust `autoplayDelay` prop as needed. Because there's no dependency on lucide-react or Tailwind for the card internals, this integrates cleanly into non-Tailwind or non-shadcn projects with minimal adaptation — one of the more portable stashed components.

```tsx
"use client";

import React, { useState, useEffect, useCallback, useRef } from "react";

const ChevronLeftIcon = () => (
  <svg width="20" height="20" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={2.5}>
    <path strokeLinecap="round" strokeLinejoin="round" d="M15 19l-7-7 7-7" />
  </svg>
);

const ChevronRightIcon = () => (
  <svg width="20" height="20" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={2.5}>
    <path strokeLinecap="round" strokeLinejoin="round" d="M9 5l7 7-7 7" />
  </svg>
);

const ArrowRightIcon = () => (
  <svg width="13" height="13" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={2.5}>
    <path strokeLinecap="round" strokeLinejoin="round" d="M14 5l7 7m0 0l-7 7m7-7H3" />
  </svg>
);

export interface CarouselItem {
  tag?: string;
  titleLine1: string;
  titleLine2?: string;
  desc?: string;
  img: string;
  ctaText?: string;
  ctaUrl?: string;
}

export interface CoverFlowCarouselProps {
  items?: CarouselItem[];
  sectionLabel?: string;
  autoplay?: boolean;
  autoplayDelay?: number;
  className?: string;
  onCtaClick?: (item: CarouselItem) => void;
}

export const defaultDishes: CarouselItem[] = [
  {
    tag: "#Signature",
    titleLine1: "BUTTER CHICKEN",
    titleLine2: "– DELHI HERITAGE",
    desc: "Velvety roasted tomato and fenugreek gravy with tender charred chicken",
    img: "/* project image */",
    ctaText: "View Menu",
    ctaUrl: "#",
  },
  {
    tag: "#ChefSpecial",
    titleLine1: "TANDOORI CHOPS",
    titleLine2: "– SMOKED SPICE",
    desc: "Grass-fed lamb chops charred in live charcoal tandoor with Kashmiri spices",
    img: "/* project image */",
    ctaText: "View Menu",
    ctaUrl: "#",
  },
  {
    tag: "#Vegetarian",
    titleLine1: "PANEER TIKKA",
    titleLine2: "– CLAY ROASTED",
    desc: "Artisan cottage cheese marinated in spiced yogurt, bell peppers & saffron",
    img: "/* project image */",
    ctaText: "View Menu",
    ctaUrl: "#",
  },
  {
    tag: "#CoastalCatch",
    titleLine1: "MALABAR PRAWNS",
    titleLine2: "– COCONUT GRAVY",
    desc: "Jumbo wild tiger prawns simmered in fragrant curry leaves and coconut milk",
    img: "/* project image */",
    ctaText: "View Menu",
    ctaUrl: "#",
  },
  {
    tag: "#ArtisanBake",
    titleLine1: "TRUFFLE NAAN",
    titleLine2: "– CHARCOAL OVEN",
    desc: "Crispy puffed leavened bread brushed with pure ghee and black winter truffle",
    img: "/* project image */",
    ctaText: "View Menu",
    ctaUrl: "#",
  },
];

export function CoverFlowCarousel({
  items = defaultDishes,
  sectionLabel = "BEST SELLERS",
  autoplay = true,
  autoplayDelay = 5000,
  className = "",
  onCtaClick,
}: CoverFlowCarouselProps) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [isHovered, setIsHovered] = useState(false);
  const touchStartX = useRef(0);
  const total = items.length;

  const nextSlide = useCallback(() => {
    setCurrentIndex((prev) => (prev + 1) % total);
  }, [total]);

  const prevSlide = useCallback(() => {
    setCurrentIndex((prev) => (prev - 1 + total) % total);
  }, [total]);

  const goToSlide = (idx: number) => {
    setCurrentIndex(idx % total);
  };

  useEffect(() => {
    if (!autoplay || isHovered || total <= 1) return;
    const interval = setInterval(nextSlide, autoplayDelay);
    return () => clearInterval(interval);
  }, [autoplay, autoplayDelay, isHovered, nextSlide, total]);

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "ArrowLeft") prevSlide();
      if (e.key === "ArrowRight") nextSlide();
    };
    window.addEventListener("keydown", handleKeyDown);
    return () => window.removeEventListener("keydown", handleKeyDown);
  }, [nextSlide, prevSlide]);

  const handleTouchStart = (e: React.TouchEvent) => {
    touchStartX.current = e.touches[0].clientX;
  };

  const handleTouchEnd = (e: React.TouchEvent) => {
    const diff = e.changedTouches[0].clientX - touchStartX.current;
    if (Math.abs(diff) > 45) {
      if (diff < 0) nextSlide();
      else prevSlide();
    }
  };

  if (!items || items.length === 0) return null;

  return (
    <section
      className={`relative w-full min-h-[760px] flex items-center justify-center overflow-hidden py-12 select-none ${className}`}
      style={{
        backgroundColor: "#0c0a09",
        color: "#ffffff",
        fontFamily: "system-ui, -apple-system, sans-serif",
      }}
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
      onTouchStart={handleTouchStart}
      onTouchEnd={handleTouchEnd}
    >
      <div className="absolute inset-0 overflow-hidden pointer-events-none z-0">
        <img
          src={items[currentIndex]?.img}
          alt="ambience background"
          style={{
            width: "100%",
            height: "100%",
            objectFit: "cover",
            filter: "brightness(0.22) blur(32px)",
            transform: "scale(1.15)",
            transition: "opacity 1000ms ease, filter 1000ms ease",
          }}
        />
        <div
          className="absolute inset-0"
          style={{
            background: "radial-gradient(circle at center, rgba(12,10,9,0.3) 0%, rgba(12,10,9,0.92) 100%)",
          }}
        />
      </div>

      <div className="relative w-full max-w-6xl mx-auto px-4 z-10 flex flex-col items-center">
        {sectionLabel && (
          <div className="flex items-center gap-3 mb-8">
            <span style={{ width: "36px", height: "1px", background: "linear-gradient(90deg, transparent, #c5a880)" }} />
            <h3
              style={{
                fontSize: "0.75rem",
                fontWeight: 700,
                letterSpacing: "0.3em",
                textTransform: "uppercase",
                color: "#c5a880",
                margin: 0,
              }}
            >
              {sectionLabel}
            </h3>
            <span style={{ width: "36px", height: "1px", background: "linear-gradient(90deg, #c5a880, transparent)" }} />
          </div>
        )}

        <div
          className="relative w-full h-[520px] flex justify-center items-center mb-8"
          style={{ perspective: "1400px" }}
        >
          {items.map((item, idx) => {
            const offset = (idx - currentIndex + total) % total;

            let transform = "translateX(0px) scale(0.4) rotateY(0deg)";
            let opacity = 0;
            let zIndex = 0;
            let filter = "brightness(0.4) blur(2px)";
            let isCenter = false;

            if (offset === 0) {
              isCenter = true;
              transform = "translateX(0px) scale(1) rotateY(0deg)";
              opacity = 1;
              zIndex = 30;
              filter = "brightness(1)";
            } else if (offset === 1) {
              transform = "translateX(285px) scale(0.84) rotateY(-24deg)";
              opacity = 0.65;
              zIndex = 20;
              filter = "brightness(0.75)";
            } else if (offset === 2) {
              transform = "translateX(510px) scale(0.68) rotateY(-38deg)";
              opacity = 0.38;
              zIndex = 10;
              filter = "brightness(0.55) blur(1px)";
            } else if (offset === total - 1) {
              transform = "translateX(-285px) scale(0.84) rotateY(24deg)";
              opacity = 0.65;
              zIndex = 20;
              filter = "brightness(0.75)";
            } else if (offset === total - 2) {
              transform = "translateX(-510px) scale(0.68) rotateY(38deg)";
              opacity = 0.38;
              zIndex = 10;
              filter = "brightness(0.55) blur(1px)";
            }

            return (
              <div
                key={idx}
                onClick={() => !isCenter && goToSlide(idx)}
                style={{
                  position: "absolute",
                  width: "330px",
                  height: "500px",
                  borderRadius: "18px",
                  overflow: "hidden",
                  backgroundColor: "#171311",
                  border: "1px solid rgba(255, 255, 255, 0.12)",
                  transform,
                  opacity,
                  zIndex,
                  filter,
                  transformOrigin: "center center",
                  transition: "all 800ms cubic-bezier(0.25, 1, 0.5, 1)",
                  boxShadow: isCenter
                    ? "0 25px 60px rgba(0,0,0,0.9), 0 0 35px rgba(197,168,128,0.25)"
                    : "0 15px 35px rgba(0,0,0,0.5)",
                  cursor: isCenter ? "default" : "pointer",
                }}
              >
                <img
                  src={item.img}
                  alt={item.titleLine1}
                  style={{ position: "absolute", inset: 0, width: "100%", height: "100%", objectFit: "cover" }}
                />

                <div
                  style={{
                    position: "absolute",
                    inset: 0,
                    background:
                      "linear-gradient(180deg, rgba(0,0,0,0.4) 0%, rgba(0,0,0,0.1) 25%, rgba(0,0,0,0.68) 60%, rgba(0,0,0,0.96) 100%)",
                    pointerEvents: "none",
                    zIndex: 10,
                  }}
                />

                <div
                  style={{
                    position: "relative",
                    width: "100%",
                    height: "100%",
                    padding: "20px 18px 22px",
                    display: "flex",
                    flexDirection: "column",
                    justifyContent: "space-between",
                    textAlign: "center",
                    zIndex: 20,
                    opacity: isCenter ? 1 : 0,
                    transform: isCenter ? "translateY(0px)" : "translateY(16px)",
                    transition: "opacity 500ms ease, transform 500ms ease",
                    pointerEvents: isCenter ? "auto" : "none",
                  }}
                >
                  <div style={{ textAlign: "right", width: "100%", paddingRight: "4px" }}>
                    <span
                      style={{
                        display: "inline-block",
                        fontSize: "0.78rem",
                        fontWeight: 600,
                        letterSpacing: "0.06em",
                        color: "rgba(255,255,255,0.9)",
                        textShadow: "0 2px 6px rgba(0,0,0,0.8)",
                      }}
                    >
                      {item.tag}
                    </span>
                  </div>

                  <div
                    style={{
                      display: "flex",
                      flexDirection: "column",
                      alignItems: "center",
                      gap: "3px",
                      marginTop: "auto",
                      paddingBottom: "4px",
                    }}
                  >
                    <h2
                      style={{
                        fontSize: "1.65rem",
                        fontWeight: 900,
                        textTransform: "uppercase",
                        letterSpacing: "0.04em",
                        color: "#ffffff",
                        margin: 0,
                        lineHeight: 1.1,
                        textShadow: "0 3px 12px rgba(0,0,0,0.95)",
                      }}
                    >
                      {item.titleLine1}
                    </h2>

                    {item.titleLine2 && (
                      <span
                        style={{
                          fontSize: "1.1rem",
                          fontWeight: 700,
                          textTransform: "uppercase",
                          letterSpacing: "0.06em",
                          color: "#f3f0ea",
                          lineHeight: 1.2,
                          textShadow: "0 3px 10px rgba(0,0,0,0.9)",
                        }}
                      >
                        {item.titleLine2}
                      </span>
                    )}

                    <div
                      style={{
                        width: "34px",
                        height: "2px",
                        backgroundColor: "#c5a880",
                        borderRadius: "2px",
                        margin: "5px auto 4px",
                        boxShadow: "0 0 8px rgba(197,168,128,0.7)",
                      }}
                    />

                    {item.desc && (
                      <p
                        style={{
                          fontSize: "0.82rem",
                          fontStyle: "italic",
                          color: "rgba(255,255,255,0.9)",
                          maxWidth: "280px",
                          margin: "0 0 10px",
                          lineHeight: 1.3,
                          textShadow: "0 2px 8px rgba(0,0,0,0.9)",
                        }}
                      >
                        {item.desc}
                      </p>
                    )}

                    <a
                      href={item.ctaUrl || "#"}
                      onClick={(e) => {
                        if (onCtaClick) {
                          e.preventDefault();
                          onCtaClick(item);
                        }
                      }}
                      style={{
                        display: "inline-flex",
                        alignItems: "center",
                        gap: "6px",
                        padding: "7px 18px",
                        borderRadius: "9999px",
                        background: "linear-gradient(135deg, #c5a880 0%, #a48256 100%)",
                        color: "#110d0c",
                        fontSize: "0.72rem",
                        fontWeight: 800,
                        letterSpacing: "0.14em",
                        textTransform: "uppercase",
                        textDecoration: "none",
                        boxShadow: "0 4px 14px rgba(0,0,0,0.4), 0 0 15px rgba(197,168,128,0.3)",
                        cursor: "pointer",
                        transition: "transform 200ms ease, box-shadow 200ms ease",
                      }}
                    >
                      <span>{item.ctaText || "View Menu"}</span>
                      <ArrowRightIcon />
                    </a>
                  </div>
                </div>
              </div>
            );
          })}
        </div>

        <button
          onClick={prevSlide}
          aria-label="Previous dish"
          style={{
            position: "absolute",
            left: "24px",
            top: "50%",
            transform: "translateY(-50%)",
            width: "46px",
            height: "46px",
            borderRadius: "50%",
            backgroundColor: "rgba(0,0,0,0.55)",
            border: "1px solid rgba(255,255,255,0.2)",
            color: "#ffffff",
            display: "flex",
            alignItems: "center",
            justifyContent: "center",
            backdropFilter: "blur(8px)",
            cursor: "pointer",
            boxShadow: "0 8px 24px rgba(0,0,0,0.4)",
            zIndex: 40,
            transition: "all 200ms ease",
          }}
        >
          <ChevronLeftIcon />
        </button>

        <button
          onClick={nextSlide}
          aria-label="Next dish"
          style={{
            position: "absolute",
            right: "24px",
            top: "50%",
            transform: "translateY(-50%)",
            width: "46px",
            height: "46px",
            borderRadius: "50%",
            backgroundColor: "rgba(0,0,0,0.55)",
            border: "1px solid rgba(255,255,255,0.2)",
            color: "#ffffff",
            display: "flex",
            alignItems: "center",
            justifyContent: "center",
            backdropFilter: "blur(8px)",
            cursor: "pointer",
            boxShadow: "0 8px 24px rgba(0,0,0,0.4)",
            zIndex: 40,
            transition: "all 200ms ease",
          }}
        >
          <ChevronRightIcon />
        </button>

        <div style={{ display: "flex", alignItems: "center", justifyContent: "center", gap: "8px", zIndex: 30 }}>
          {items.map((_, idx) => (
            <button
              key={idx}
              onClick={() => goToSlide(idx)}
              aria-label={`Go to slide ${idx + 1}`}
              style={{
                height: "8px",
                width: idx === currentIndex ? "28px" : "8px",
                borderRadius: "9999px",
                backgroundColor: idx === currentIndex ? "#c5a880" : "rgba(255,255,255,0.25)",
                border: "none",
                cursor: "pointer",
                boxShadow: idx === currentIndex ? "0 0 10px rgba(197,168,128,0.7)" : "none",
                transition: "all 300ms ease",
              }}
            />
          ))}
        </div>
      </div>
    </section>
  );
}

export const Component = CoverFlowCarousel;
export default CoverFlowCarousel;
```

---

## Image Stream Hero
- **Category**: hero background / decorative image corridor — pure-CSS 3D perspective effect
- **Stack**: React, TypeScript, Tailwind (`cn()` util only) — the actual motion is pure CSS `@keyframes` computed once in JS and injected via a `<style>` tag; no animation library, no per-frame JS (the perspective/3D math runs once at render time to build the keyframe strings, not on every frame)
- **Deps**: none beyond React + the project's `cn()` helper
- **Summary**: A decorative "flying through a corridor of photos" hero background — two mirrored rails of image cards appear to rush from a vanishing point toward the viewer, using pure CSS 3D transforms (`perspective`, `translate3d`, `rotateY`) driven by keyframes that are mathematically derived (not eyeballed) so cards grow geometrically in apparent size as they approach, keeping the "ribbon" of cards visually continuous with no gaps or tears. Cards are born off-axis (negative `railBirth`) specifically so the center of the screen is never empty at any point in the loop — a deliberate fix for a specific visual artifact (a "hole" blinking open at center once per cycle) that's explained in the code's own extensive comments. Respects `prefers-reduced-motion` by pausing (not hiding) the animation, so it freezes as a complete, sensible-looking still frame instead of collapsing to the center. Fully responsive via container query units (`cqw`) throughout, so the whole corridor scales proportionally with its container rather than the viewport.
- **Notes**: This is an unusually well-engineered piece — the source comments explain the geometry reasoning in detail (why depth is geometric not linear, why the rails "fan" open early, why cards are born off-axis) and are worth keeping intact if this is reused, since they explain *why* the numeric defaults are what they are (described as "fitted numerically against a reference recording," not arbitrary). Practical notes for use: (1) it's purely decorative/`aria-hidden` — pass real content via the `children` prop to overlay on top (headline, CTA, etc.), the corridor itself renders no text; (2) `images` array is cycled — fewer images than `cards` (default 9) just repeats them, so a handful of images is enough, no need for a huge unique set; (3) the extensive prop-level JSDoc comments in the original source (quoted inline on `CorridorPath` fields explaining what breaks if you change them, e.g. "Raising `exitHeight`, dropping `cards`, or pulling `railExit` in all push toward a visible tear near the frame edge") are genuinely load-bearing documentation — don't strip them out when adapting, they're the only place the geometry's failure modes are explained. No demo/usage file was included with this component — when using it, pass an `images` array of `{ src, alt? }` objects and wrap page content as `children`.

```tsx
"use client";

import * as React from "react";
import { cn } from "@/lib/utils";

export type CorridorPath = {
  /** Strength of the projection. Lower is a wider-angle, more dramatic rush. @default 30 */
  perspective?: number;
  /** Card width in world units. @default 18 */
  cardWidth?: number;
  /** Card height in world units. @default 25 */
  cardHeight?: number;
  /** Corner radius applied to each card. @default 0.4 */
  cardRadius?: number;
  /** On-screen card height at the waist, where a card is born. @default 2.6 */
  birthHeight?: number;
  /** On-screen card height as a card leaves the frame. @default 46 */
  exitHeight?: number;
  /**
   * Lateral offset at birth. Negative starts the card across the axis so the
   * centre never opens up. @default -11
   */
  railBirth?: number;
  /** Lateral offset once the rails have finished opening. @default 44 */
  railExit?: number;
  /** How front-loaded the opening is. >1 opens early then holds. @default 3.3 */
  fan?: number;
  /** Y-rotation at birth, degrees. @default 6 */
  turnBirth?: number;
  /** Y-rotation at exit, degrees. @default 28 */
  turnExit?: number;
  /** Keyframe stops used to trace the curve. Raise only if motion looks faceted. @default 24 */
  stops?: number;
};

const PATH: Required<CorridorPath> = {
  perspective: 30,
  cardWidth: 18,
  cardHeight: 25,
  cardRadius: 0.4,
  birthHeight: 2.6,
  exitHeight: 46,
  railBirth: -11,
  railExit: 44,
  fan: 3.3,
  turnBirth: 6,
  turnExit: 28,
  stops: 24,
};

/** Sample the path once so the CSS keyframes trace the real curve. */
function keyframes(dir: 1 | -1, name: string, p: Required<CorridorPath>) {
  const steps: string[] = [];
  for (let s = 0; s <= p.stops; s++) {
    const u = s / p.stops;
    const scale =
      (p.birthHeight / p.cardHeight) *
      Math.pow(p.exitHeight / p.birthHeight, u);
    const z = p.perspective * (1 - 1 / scale);
    const rail =
      p.railExit - (p.railExit - p.railBirth) * Math.pow(1 - u, p.fan);
    const turn = p.turnBirth + (p.turnExit - p.turnBirth) * u;
    steps.push(
      `${(u * 100).toFixed(2)}%{transform:translate3d(${(dir * rail).toFixed(
        2,
      )}cqw,0,${z.toFixed(2)}cqw) rotateY(${(-dir * turn).toFixed(2)}deg)}`,
    );
  }
  return `@keyframes ${name}{${steps.join("")}}`;
}

export type StreamImage = {
  src: string;
  alt?: string;
};

export type ImageStreamHeroProps = {
  images: StreamImage[];
  /** @default 9 */
  cards?: number;
  /** @default 18 */
  speed?: number;
  /** @default 55 */
  axis?: number;
  path?: CorridorPath;
  children?: React.ReactNode;
  className?: string;
};

export function ImageStreamHero({
  images,
  cards = 9,
  speed = 18,
  axis = 55,
  path,
  children,
  className,
  ...props
}: React.ComponentProps<"div"> & ImageStreamHeroProps) {
  const id = React.useId().replace(/[^a-zA-Z0-9]/g, "");
  const right = `ish-r-${id}`;
  const left = `ish-l-${id}`;
  const card = `ish-c-${id}`;

  const p = React.useMemo(() => ({ ...PATH, ...path }), [path]);

  const css = React.useMemo(
    () =>
      `${keyframes(1, right, p)}${keyframes(-1, left, p)}` +
      `@media(prefers-reduced-motion:reduce){.${card}{animation-play-state:paused}}`,
    [right, left, card, p],
  );

  return (
    <div
      className={cn("relative overflow-hidden", className)}
      {...props}
      style={{ containerType: "inline-size", ...props.style }}
    >
      <style>{css}</style>

      <div
        aria-hidden
        className="pointer-events-none absolute inset-0"
        style={{
          perspective: `${p.perspective}cqw`,
          perspectiveOrigin: `50% ${axis}%`,
        }}
      >
        <div
          className="absolute inset-0"
          style={{ transformStyle: "preserve-3d" }}
        >
          {[right, left].map((name) =>
            Array.from({ length: cards }, (_, i) => {
              const img = images[i % Math.max(images.length, 1)];
              return (
                <div
                  key={`${name}-${i}`}
                  className={cn(card, "absolute overflow-hidden")}
                  style={{
                    left: "50%",
                    top: `${axis}%`,
                    width: `${p.cardWidth}cqw`,
                    height: `${p.cardHeight}cqw`,
                    marginLeft: `${-p.cardWidth / 2}cqw`,
                    marginTop: `${-p.cardHeight / 2}cqw`,
                    borderRadius: `${p.cardRadius}cqw`,
                    animation: `${name} ${speed}s linear infinite`,
                    animationDelay: `${-(i * speed) / cards}s`,
                    backfaceVisibility: "hidden",
                  }}
                >
                  {img ? (
                    <img
                      src={img.src}
                      alt={img.alt ?? ""}
                      loading="lazy"
                      decoding="async"
                      className="h-full w-full object-cover"
                      draggable={false}
                    />
                  ) : null}
                </div>
              );
            }),
          )}
        </div>
      </div>

      {children}
    </div>
  );
}

export default ImageStreamHero;
```

**Usage** (no demo file was included with this component — minimal example):
```tsx
import { ImageStreamHero } from "@/components/ui/image-stream-hero";

export default function Hero() {
  return (
    <ImageStreamHero
      images={[
        { src: "/* project image 1 */" },
        { src: "/* project image 2 */" },
        { src: "/* project image 3 */" },
      ]}
      className="h-screen w-full bg-black"
    >
      <div className="relative z-10 flex h-full items-center justify-center">
        <h1 className="text-white text-5xl font-bold">Your headline here</h1>
      </div>
    </ImageStreamHero>
  );
}
```

---

<!-- Add new components below this line, following the same format -->
