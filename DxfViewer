import { fromEvent } from 'rxjs';
import * as THREE from 'three';
import { OrbitControls } from '../../shared/define/OrbitControls.js';

export class DxfViewer {
    public parent: HTMLElement;
    public scene: THREE.Scene;
    public renderer: THREE.WebGLRenderer;
    public camera: THREE.OrthographicCamera;
    public controls: OrbitControls;
    public font: THREE.Font;

    constructor(data: any, parent: HTMLElement, width?: number, height?: number, font?: THREE.Font) {
        this.parent = parent;
        this.font = font;
        fromEvent(this.parent, 'resize').subscribe((event) => this.onresize(width, height));
        fromEvent(this.parent, 'click').subscribe((event: MouseEvent) => this.onclick(event));
        this.createLineTypeShaders(data);
        this.scene = new THREE.Scene();
        // Create scene from dxf object (data)
        let entity, obj;
        const dims = {
            min: { x: Number.NaN, y: Number.NaN, z: Number.NaN },
            max: { x: Number.NaN, y: Number.NaN, z: Number.NaN },
        };
        for (let i = 0; i < data.entities.length; i++) {
            entity = data.entities[i];

            if (entity.type === 'DIMENSION') {
                if (entity.block) {
                    const block = data.blocks[entity.block];
                    if (!block) {
                        console.error(`Missing referenced block ${entity.block}`);
                        continue;
                    }
                    for (let j = 0; j < block.entities.length; j++) {
                        obj = this.drawEntity(block.entities[j], data);
                    }
                } else {
                    console.log('WARNING: No block for DIMENSION entity');
                }
            } else {
                obj = this.drawEntity(entity, data);
            }

            if (obj) {
                const bbox = new THREE.Box3().setFromObject(obj);
                if (bbox.min.x && (Number.isNaN(dims.min.x) || (dims.min.x > bbox.min.x))) { dims.min.x = bbox.min.x; }
                if (bbox.min.y && (Number.isNaN(dims.min.y) || (dims.min.y > bbox.min.y))) { dims.min.y = bbox.min.y; }
                if (bbox.min.z && (Number.isNaN(dims.min.z) || (dims.min.z > bbox.min.z))) { dims.min.z = bbox.min.z; }
                if (bbox.max.x && (Number.isNaN(dims.max.x) || (dims.max.x < bbox.max.x))) { dims.max.x = bbox.max.x; }
                if (bbox.max.y && (Number.isNaN(dims.max.y) || (dims.max.y < bbox.max.y))) { dims.max.y = bbox.max.y; }
                if (bbox.max.z && (Number.isNaN(dims.max.z) || (dims.max.z < bbox.max.z))) { dims.max.z = bbox.max.z; }
                this.scene.add(obj);
            }
            obj = null;
        }

        width = width || this.parent.clientWidth;
        height = height || this.parent.clientHeight;
        const aspectRatio = width / height;

        const upperRightCorner = { x: dims.max.x, y: dims.max.y };
        const lowerLeftCorner = { x: dims.min.x, y: dims.min.y };

        // Figure out the current viewport extents
        let vp_width = upperRightCorner.x - lowerLeftCorner.x;
        let vp_height = upperRightCorner.y - lowerLeftCorner.y;
        const center = {
            x: vp_width / 2 + lowerLeftCorner.x,
            y: vp_height / 2 + lowerLeftCorner.y
        };

        // Fit all objects into current ThreeDXF viewer
        const extentsAspectRatio = Math.abs(vp_width / vp_height);
        if (aspectRatio > extentsAspectRatio) {
            vp_width = vp_height * aspectRatio;
        } else {
            vp_height = vp_width / aspectRatio;
        }

        const viewPort = {
            bottom: -vp_height / 2,
            left: -vp_width / 2,
            top: vp_height / 2,
            right: vp_width / 2,
            center: {
                x: center.x,
                y: center.y
            }
        };

        this.camera = new THREE.OrthographicCamera(viewPort.left, viewPort.right, viewPort.top, viewPort.bottom, 1, 19);
        this.camera.position.z = 10;
        this.camera.position.x = viewPort.center.x;
        this.camera.position.y = viewPort.center.y;

        this.renderer = new THREE.WebGLRenderer();
        this.renderer.setSize(width, height);
        this.renderer.setClearColor(0xfffffff, 1);

        this.parent.append(this.renderer.domElement);
        this.parent.hidden = false;

        this.controls = new OrbitControls(this.camera, parent);
        this.controls.target.x = this.camera.position.x;
        this.controls.target.y = this.camera.position.y;
        this.controls.target.z = 0;
        this.controls.enableRotate = false;
        this.controls.zoomSpeed = 3;

        // Uncommend this to disable rotation (does not make much sense with 2D drawings).
        // controls.enableRotate = false;
        fromEvent(this.controls, 'change').subscribe((event) => this.render());
        // this.controls.addEventListener('change', this.render);
        this.render();
        this.controls.update();

    }

    onclick(event) {
        const el = this.renderer.domElement;

        const vector = new THREE.Vector3(
            ((event.pageX - el.clientLeft) / el.clientWidth) * 2 - 1,
            -((event.pageY - el.clientTop) / el.clientHeight) * 2 + 1,
            0.5);
        vector.unproject(this.camera);

        const dir = vector.sub(this.camera.position).normalize();

        const distance = -this.camera.position.z / dir.z;

        const pos = this.camera.position.clone().add(dir.multiplyScalar(distance));

        console.log(pos.x, pos.y); // Position in cad that is clicked
    }

    onresize(width, height) {
        const originalWidth = this.renderer.domElement.width;
        const originalHeight = this.renderer.domElement.height;

        const hscale = width / originalWidth;
        const vscale = height / originalHeight;

        this.camera.top = (vscale * this.camera.top);
        this.camera.bottom = (vscale * this.camera.bottom);
        this.camera.left = (hscale * this.camera.left);
        this.camera.right = (hscale * this.camera.right);

        //        camera.updateProjectionMatrix();

        this.renderer.setSize(width, height);
        this.renderer.setClearColor(0xfffffff, 1);
        this.render();
    }
    render() {
        this.renderer.render(this.scene, this.camera);
    }

    drawEntity(entity: any, data: any): THREE.Object3D {
        let mesh: THREE.Object3D;
        if (entity.type === 'CIRCLE' || entity.type === 'ARC') {
            mesh = this.drawArc(entity, data);
        } else if (entity.type === 'LWPOLYLINE' || entity.type === 'LINE' || entity.type === 'POLYLINE') {
            mesh = this.drawLine(entity, data);
        } else if (entity.type === 'TEXT') {
            mesh = this.drawText(entity, data);
        } else if (entity.type === 'SOLID') {
            mesh = this.drawSolid(entity, data);
        } else if (entity.type === 'POINT') {
            mesh = this.drawPoint(entity, data);
        } else if (entity.type === 'INSERT') {
            mesh = this.drawBlock(entity, data);
        } else if (entity.type === 'SPLINE') {
            mesh = this.drawSpline(entity, data);
        } else if (entity.type === 'MTEXT') {
            mesh = this.drawMtext(entity, data);
        } else if (entity.type === 'ELLIPSE') {
            mesh = this.drawEllipse(entity, data);
        } else {
            console.log(`Unsupported Entity Type: ${entity.type}`);
        }
        return mesh;
    }

    drawEllipse(entity: any, data: any): THREE.Line {
        const color = this.getColor(entity, data);
        const xrad = Math.sqrt(Math.pow(entity.majorAxisEndPoint.x, 2) + Math.pow(entity.majorAxisEndPoint.y, 2));
        const yrad = xrad * entity.axisRatio;
        const rotation = Math.atan2(entity.majorAxisEndPoint.y, entity.majorAxisEndPoint.x);

        const curve = new THREE.EllipseCurve(
            entity.center.x, entity.center.y,
            xrad, yrad,
            entity.startAngle, entity.endAngle,
            false, // Always counterclockwise
            rotation
        );

        const points = curve.getPoints(50);
        const geometry = new THREE.BufferGeometry().setFromPoints(points);
        const material = new THREE.LineBasicMaterial({ linewidth: 1, color: color.getHex() });

        // Create the final object to add to the scene
        const ellipse = new THREE.Line(geometry, material);
        return ellipse;
    }

    drawMtext(entity, data): THREE.Mesh {
        const color = this.getColor(entity, data);
        const geometry = new THREE.TextGeometry(entity.text, {
            font: this.font,
            size: entity.height * (4 / 5),
            height: 1,
            curveSegments: 12,
            bevelEnabled: false,
            bevelThickness: 10,
            bevelSize: 8,
        });


        const material = new THREE.MeshBasicMaterial({ color: color.getHex() });
        const text = new THREE.Mesh(geometry, material);

        // Measure what we rendered.
        const measure = new THREE.Box3();
        measure.setFromObject(text);

        const textWidth = measure.max.x - measure.min.x;

        // If the text ends up being wider than the box, it's supposed
        // to be multiline. Doing that in threeJS is overkill.
        if (textWidth > entity.width) {
            console.log(`Can't render this multipline MTEXT ${entity.text}, sorry.`);
             return null;
        }

        text.position.z = 0;
        switch (entity.attachmentPoint) {
            case 1:
                // Top Left
                text.position.x = entity.position.x;
                text.position.y = entity.position.y - entity.height;
                break;
            case 2:
                // Top Center
                text.position.x = entity.position.x - textWidth / 2;
                text.position.y = entity.position.y - entity.height;
                break;
            case 3:
                // Top Right
                text.position.x = entity.position.x - textWidth;
                text.position.y = entity.position.y - entity.height;
                break;

            case 4:
                // Middle Left
                text.position.x = entity.position.x;
                text.position.y = entity.position.y - entity.height / 2;
                break;
            case 5:
                // Middle Center
                text.position.x = entity.position.x - textWidth / 2;
                text.position.y = entity.position.y - entity.height / 2;
                break;
            case 6:
                // Middle Right
                text.position.x = entity.position.x - textWidth;
                text.position.y = entity.position.y - entity.height / 2;
                break;

            case 7:
                // Bottom Left
                text.position.x = entity.position.x;
                text.position.y = entity.position.y;
                break;
            case 8:
                // Bottom Center
                text.position.x = entity.position.x - textWidth / 2;
                text.position.y = entity.position.y;
                break;
            case 9:
                // Bottom Right
                text.position.x = entity.position.x - textWidth;
                text.position.y = entity.position.y;
                break;

            default:
                return null;
        }

        return text;
    }

    drawSpline(entity, data): THREE.Line {
        const color = this.getColor(entity, data);
        const points = entity.controlPoints.map(function (vec) {
            return new THREE.Vector2(vec.x, vec.y);
        });

        let interpolatedPoints = [];
        if (entity.degreeOfSplineCurve === 2 || entity.degreeOfSplineCurve === 3) {
            for (let i = 0; i + 2 < points.length; i = i + 2) {
                if (entity.degreeOfSplineCurve === 2) {
                    const curve = new THREE.QuadraticBezierCurve(points[i], points[i + 1], points[i + 2]);
                    interpolatedPoints.push.apply(interpolatedPoints, curve.getPoints(50));
                } else {
                    const curve = new THREE.QuadraticBezierCurve3(points[i], points[i + 1], points[i + 2]);
                    interpolatedPoints.push.apply(interpolatedPoints, curve.getPoints(50));
                }
            }
        } else {
            const curve = new THREE.SplineCurve(points);
            interpolatedPoints = curve.getPoints(100);
        }

        const geometry = new THREE.BufferGeometry().setFromPoints(interpolatedPoints);
        const material = new THREE.LineBasicMaterial({ linewidth: 1, color: color.getHex() });
        const splineObject = new THREE.Line(geometry, material);

        return splineObject;
    }

    drawLine(entity, data): THREE.Line {
        const geometry = new THREE.Geometry();
        const color = this.getColor(entity, data);
        let material, lineType, vertex, startPoint, endPoint, bulgeGeometry,
            bulge, i, line;

        // create geometry
        for (i = 0; i < entity.vertices.length; i++) {

            if (entity.vertices[i].bulge) {
                bulge = entity.vertices[i].bulge;
                startPoint = entity.vertices[i];
                endPoint = i + 1 < entity.vertices.length ? entity.vertices[i + 1] : geometry.vertices[0];

                bulgeGeometry = new BulgeGeometry(startPoint, endPoint, bulge, 10);

                geometry.vertices.push.apply(geometry.vertices, bulgeGeometry.vertices);
            } else {
                vertex = entity.vertices[i];
                geometry.vertices.push(new THREE.Vector3(vertex.x, vertex.y, 0));
            }

        }
        if (entity.shape) {
            geometry.vertices.push(geometry.vertices[0]);
        }

        // set material
        if (entity.lineType) {
            lineType = data.tables.lineType.lineTypes[entity.lineType];
        }

        if (lineType && lineType.pattern && lineType.pattern.length !== 0) {
            material = new THREE.LineDashedMaterial({ color: color.getHex(), gapSize: 4, dashSize: 4 });
        } else {
            material = new THREE.LineBasicMaterial({ linewidth: 1, color: color.getHex() });
        }

        // if(lineType && lineType.pattern && lineType.pattern.length !== 0) {

        //           geometry.computeLineDistances();

        //           // Ugly hack to add diffuse to this. Maybe copy the uniforms object so we
        //           // don't add diffuse to a material.
        //           lineType.material.uniforms.diffuse = { type: 'c', value: new THREE.Color(color) };

        // 	material = new THREE.ShaderMaterial({
        // 		uniforms: lineType.material.uniforms,
        // 		vertexShader: lineType.material.vertexShader,
        // 		fragmentShader: lineType.material.fragmentShader
        // 	});
        // }else {
        // 	material = new THREE.LineBasicMaterial({ linewidth: 1, color: color });
        // }

        line = new THREE.Line(geometry, material);
        return line;
    }

    drawArc(entity, data): THREE.Line {
        let startAngle, endAngle;
        if (entity.type === 'CIRCLE') {
            startAngle = entity.startAngle || 0;
            endAngle = startAngle + 2 * Math.PI;
        } else {
            startAngle = entity.startAngle;
            endAngle = entity.endAngle;
        }

        const curve = new THREE.ArcCurve(0, 0, entity.radius, startAngle, endAngle, true);

        const points = curve.getPoints(32);
        const geometry = new THREE.BufferGeometry().setFromPoints(points);

        const material = new THREE.LineBasicMaterial({ color: this.getColor(entity, data).getHex() });

        const arc = new THREE.Line(geometry, material);
        arc.position.x = entity.center.x;
        arc.position.y = entity.center.y;
        arc.position.z = entity.center.z;

        return arc;
    }

    drawSolid(entity, data): THREE.Mesh {
        const geometry = new THREE.Geometry();

        const verts = geometry.vertices;
        verts.push(new THREE.Vector3(entity.points[0].x, entity.points[0].y, entity.points[0].z));
        verts.push(new THREE.Vector3(entity.points[1].x, entity.points[1].y, entity.points[1].z));
        verts.push(new THREE.Vector3(entity.points[2].x, entity.points[2].y, entity.points[2].z));
        verts.push(new THREE.Vector3(entity.points[3].x, entity.points[3].y, entity.points[3].z));

        // Calculate which direction the points are facing (clockwise or counter-clockwise)
        const vector1 = new THREE.Vector3();
        const vector2 = new THREE.Vector3();
        vector1.subVectors(verts[1], verts[0]);
        vector2.subVectors(verts[2], verts[0]);
        vector1.cross(vector2);

        // If z < 0 then we must draw these in reverse order
        if (vector1.z < 0) {
            geometry.faces.push(new THREE.Face3(2, 1, 0));
            geometry.faces.push(new THREE.Face3(2, 3, 1));
        } else {
            geometry.faces.push(new THREE.Face3(0, 1, 2));
            geometry.faces.push(new THREE.Face3(1, 3, 2));
        }


        const material = new THREE.MeshBasicMaterial({ color: this.getColor(entity, data).getHex() });

        return new THREE.Mesh(geometry, material);

    }

    drawText(entity, data): THREE.Mesh {
        if (!this.font === null) {
            console.warn('Text is not supported without a Three.js font loaded with THREE.FontLoader!');
            return null;
        }
        const geometry = new THREE.TextGeometry(entity.text, {
            font: this.font,
            size: entity.textHeight || 12,
            height: 0,
            curveSegments: 10,
            bevelEnabled: false,
            bevelThickness: 10,
            bevelSize: 8,
        });

        const material = new THREE.MeshBasicMaterial({ color: this.getColor(entity, data).getHex() });

        const text = new THREE.Mesh(geometry, material);
        text.position.x = entity.startPoint.x;
        text.position.y = entity.startPoint.y;
        text.position.z = entity.startPoint.z;
        return text;
    }

    drawPoint(entity, data): THREE.Points {
        const geometry = new THREE.Geometry();

        geometry.vertices.push(new THREE.Vector3(entity.position.x, entity.position.y, entity.position.z));

        // TODO: could be more efficient. PointCloud per layer?
        const color = this.getColor(entity, data);

        geometry.colors = [color];
        geometry.computeBoundingBox();

        const material = new THREE.PointsMaterial({ size: 0.05, vertexColors: THREE.VertexColors });
        const point = new THREE.Points(geometry, material);
        this.scene.add(point);
        return point;
        // return null;
    }

    drawBlock(entity, data): THREE.Object3D {
        const block = data.blocks[entity.name];

        if (!block.entities) {
            return null;
        }

        const group = new THREE.Object3D();

        if (entity.xScale) { group.scale.x = entity.xScale; }
        if (entity.yScale) { group.scale.y = entity.yScale; }

        if (entity.rotation) {
            group.rotation.z = entity.rotation * Math.PI / 180;
        }

        if (entity.position) {
            group.position.x = entity.position.x;
            group.position.y = entity.position.y;
            group.position.z = entity.position.z;
        }

        for (let i = 0; i < block.entities.length; i++) {
            const childEntity = this.drawEntity(block.entities[i], data);
            if (childEntity) { group.add(childEntity); }
        }

        return group;
    }

    getColor(entity, data): THREE.Color {
        let color = 0x000000; // default
        if (entity.color) {
            color = entity.color;
        } else if (data.tables && data.tables.layer && data.tables.layer.layers[entity.layer]) {
            color = data.tables.layer.layers[entity.layer].color;
        }

        if (color == null || color === 0xffffff) {
            color = 0x000000;
        }
        return new THREE.Color(color);
    }

    createLineTypeShaders(data): void {
        if (!data.tables || !data.tables.lineType) { return; }
        const ltypes = data.tables.lineType.lineTypes;
        for (const ltype of ltypes) {
            if (!ltype.pattern) {
                continue;
            }
            ltype.material = this.createDashedLineShader(ltype.pattern);
        }
    }

    createDashedLineShader(pattern: []) {
        let totalLength = 0.0;
        const dashedLineShader = { uniforms: null, vertexShader: null, fragmentShader: null };
        for (let i = 0; i < pattern.length; i++) {
            totalLength += Math.abs(pattern[i]);
        }

        dashedLineShader.uniforms = THREE.UniformsUtils.merge([

            THREE.UniformsLib['common'],
            THREE.UniformsLib['fog'],

            {
                'pattern': { type: 'fv1', value: pattern },
                'patternLength': { type: 'f', value: totalLength }
            }

        ]);

        dashedLineShader.vertexShader = [
            'attribute float lineDistance;',

            'constying float vLineDistance;',

            THREE.ShaderChunk['color_pars_vertex'],

            'void main() {',

            THREE.ShaderChunk['color_vertex'],

            'vLineDistance = lineDistance;',

            'gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );',

            '}'
        ].join('\n');

        dashedLineShader.fragmentShader = [
            'uniform vec3 diffuse;',
            'uniform float opacity;',

            'uniform float pattern[' + pattern.length + '];',
            'uniform float patternLength;',

            'constying float vLineDistance;',

            THREE.ShaderChunk['color_pars_fragment'],
            THREE.ShaderChunk['fog_pars_fragment'],

            'void main() {',

            'float pos = mod(vLineDistance, patternLength);',

            'for ( int i = 0; i < ' + pattern.length + '; i++ ) {',
            'pos = pos - abs(pattern[i]);',
            'if( pos < 0.0 ) {',
            'if( pattern[i] > 0.0 ) {',
            'gl_FragColor = vec4(1.0, 0.0, 0.0, opacity );',
            'break;',
            '}',
            'discard;',
            '}',

            '}',

            THREE.ShaderChunk['color_fragment'],
            THREE.ShaderChunk['fog_fragment'],

            '}'
        ].join('\n');

        return dashedLineShader;
    }

    findExtents(scene: THREE.Scene) {
        let minX, maxX, minY, maxY;
        for (const child of scene.children) {
            if (child.position) {
                minX = Math.min(child.position.x, minX);
                minY = Math.min(child.position.y, minY);
                maxX = Math.max(child.position.x, maxX);
                maxY = Math.max(child.position.y, maxY);
            }
        }

        return { min: { x: minX, y: minY }, max: { x: maxX, y: maxY } };
    }

}


export class BulgeGeometry extends THREE.Geometry {

    constructor(startPoint: { x: number, y: number }, endPoint: { x: number, y: number }, bulge: number, segments: number) {
        super();
        let vertex, i,
            center, p0, p1, angle,
            radius, startAngle,
            thetaAngle;

        // const gmy = new THREE.Geometry();
        const startvector = p0 = startPoint ? new THREE.Vector2(startPoint.x, startPoint.y) : new THREE.Vector2(0, 0);
        const endvector = p1 = endPoint ? new THREE.Vector2(endPoint.x, endPoint.y) : new THREE.Vector2(1, 0);
        bulge = bulge = bulge || 1;

        angle = 4 * Math.atan(bulge);
        radius = p0.distanceTo(p1) / 2 / Math.sin(angle / 2);
        center = this.polar(startvector, radius, this.angle2(p0, p1) + (Math.PI / 2 - angle / 2));
        // By default want a segment roughly every 10 degrees
        segments = segments = segments || Math.max(Math.abs(Math.ceil(angle / (Math.PI / 18))), 6);
        startAngle = this.angle2(center, p0);
        thetaAngle = angle / segments;
        this.vertices.push(new THREE.Vector3(p0.x, p0.y, 0));
        for (i = 1; i <= segments - 1; i++) {
            vertex = this.polar(center, Math.abs(radius), startAngle + thetaAngle * i);
            this.vertices.push(new THREE.Vector3(vertex.x, vertex.y, 0));
        }
    }

    /**
 * Returns the angle in radians of the vector (p1,p2). In other words, imagine
 * putting the base of the vector at coordinates (0,0) and finding the angle
 * from vector (1,0) to (p1,p2).
 */
    public angle2(p1: { x: number, y: number }, p2: { x: number, y: number }) {
        const v1 = new THREE.Vector2(p1.x, p1.y);
        const v2 = new THREE.Vector2(p2.x, p2.y);
        v2.sub(v1); // sets v2 to be our chord
        v2.normalize();
        if (v2.y < 0) {
            return -Math.acos(v2.x);
        } else {
            return Math.acos(v2.x);
        }
    }

    public polar(point: { x: number, y: number }, distance: number, angle: number) {
        const result = { x: 0, y: 0 };
        result.x = point.x + distance * Math.cos(angle);
        result.y = point.y + distance * Math.sin(angle);
        return result;
    }
}
